This is a guide on create a custom type.  In this example we are creating a few event items for block 3707.

Item Description: The event item will drop in special blocks, and are usable.  Using the item would cost x amount of shitcoins, and get some good reward.

1. Create the configs
    Let’s create some items in the shopitems.txt (in the miningtest folder)

    ```
    id=2007|typename=coupon|cap=10|desc=此人可以帮你调查商店是否有非法勾当 顺便顺走一件物品（下次购买不要钱)|symbol=🧔|shortname=艾虎币艾|alias=carl|stat=0
    id=2008|typename=eventcoin|cap=10|desc=锁韭菜的仓 再画个大饼 10倍几率割韭菜 （直接使用 消耗10个空气币）|symbol=🥕|shortname=三爷令牌|alias=three|lifetime=1440|stat=10
    id=2009|typename=eventcoin|cap=10|desc=先洗脑 再割韭菜 几率10倍 收获加倍（直接使用 消耗20个空气币）|symbol=🎂|shortname=小赖老师教材|alias=study|lifetime=1440|stat=20
    id=2010|typename=eventcoin|cap=10|desc=2017年最热门游戏 100% 割韭菜 只割大的 （直接使用 消耗30个空气币）|symbol=🍴|shortname=分叉大法|alias=fork|lifetime=1440|stat=30
    id=2011|typename=eventcoin|cap=10|desc=提供网上钱包服务 客户的就是你的 别客气 （直接使用 消耗50个空气币）|symbol=💼|shortname=网上钱包|alias=mybitcoins|lifetime=1440|stat=50
    ```
1. About the config
    The item 2007 is the final price for the event, item type “coupon” is already defined, so we don’t need to make a class.  The rest of them use a new type “eventcoin”, which need to be coded.  Properties Used by Base Class:
    Id – must be unique
    Typename – each typename has a corresponding class to execute the functions
    Cap – this defines how many one can buy
    Desc – just a string to describe the item
    Symbol,shortname,alias – these are the strings can be used with /use command, otherwise, just for displaying
    Lifetime – if not 0, # of blocks later, this item will disappear
    Stat,maxstat – these two are not used in base class, you can use it for whatever way.  Additional property can be defined, which will be included in later 

1. Create the class
      We can create the class in an existing project, or create a new project.  Assembly name must starts with “nano.” For the api to load it.

      In this example, we will put in “Nano.Club.Thirdparty” project
      Right click the project->add->class->name it ItemEventCoin
      In the class, override the two abstract functions:

      ```
      public class ItemEventCoin : Miner.ShopItemBase
          {
              public override ShopItemBase Clone()
              {
                  return new ItemEventCoin();
              }

              public override string TypeName()
              {
                  return "eventcoin";
              }
      }
      ```
1. Let’s test them in console
    In miningtest/minerstat.txt, modify BOB’s inventory:
    id=12346|name=BOB|balance=0|luck=0||itemkey=2008,1,2009,1,2010,1,2011,1|tier=1|minerlevel=0
    Go to TestConsole/program.cs, after fakeuser creations, let’s add
    chat(Bob, null, "/coin");
    chat(Bob, null, "/item 2008");
    
    run

1. Property
    In this example we are using property “stat” to describe how many shitcoins are to be used with this item.
    Let’s test this in “UseItem” to see if it’s set correctly
    ```
            public override string UseItem(Miner.Miner master, string user, string target)
            {
                return String.Format("Coins to spend = {0}", this.GetStat());
            }
    ```
    We can run and see the result in console

1. Deduct coins
    First part of the execution is to deduct all the coins spent
    ```
    public override string UseItem(Miner.Miner master, string user, string target)
            {

                uint shitcoinid = 1002;
                MinerStat stat = master.GetStat(user);
                if (stat == null)
                    return null;

                //make sure we have enough shit coins, we don't need check this coin itself, as api checks for you
                if (stat.GetItemCount(shitcoinid) < this.GetStat())
                    return "You don't have enough coins";

                stat.ChangeAmount(shitcoinid, -this.GetStat(), true);
                stat.ChangeAmount(this.ID(), -1, true);

                master.WriteBalance(); // make sure save at the end
                return null;

            }
    ```
    Use the following console code to test:
    ```
    chat(Bob, null, "/coin");
    chat(Bob, null, "/use 2008");
    chat(Bob, null, "/coin");
    ```
    You should see the balance change 


1. Let’s create a rolling table for reward
    NOTE: this section is all helper classes, with nothing to do with the API.   Therefore you don’t have to go through it if you just want to copy/paste and see how it works
    First, we need create a configuration for rolling, let’s call it “rolling.txt” and place it in the miningtest folder.
    (Note: anything starts with # is comment)
    ```
    #name: name of the table
    #rates: roll 1000-sided dice, value=result,reward,count....repeat....
    name=shitcoin|rates=1,1013,1,5,1004,1,20,1003,1,80,1012,1
    name=eventone|rates=10,1013,1,50,1004,1,200,1003,1,800,1012,1
    name=eventtwo|rates=10,1013,2,50,1004,2,200,1003,2,800,1012,2
    name=eventthree|rates=10,1013,1,50,1004,1,200,1003,1
    name=eventfour|rates=1000,2007,1
    ```
    Second, we create the code to load this config
    ```
        public class RollingResult
        {
            public int required;
            public uint item;
            public int count;
        }

        public class RollingTable
        {
            //the following two members must match the names used in config file
            public string name;
            public string rates;


            SortedDictionary<int, RollingResult> table = new SortedDictionary<int, RollingResult>();
            Random random = new Random();

            public void Init()
            {
                string[] splits = rates.Split(',');
                for(int i = 0;i < splits.Length /3; i++)
                {
                    int required = int.Parse(splits[i*3]);
                    uint item = uint.Parse(splits[i * 3 + 1]);
                    int count = int.Parse(splits[i * 3 + 2]);

                    RollingResult result = new RollingResult();
                    result.required = required;
                    result.item = item;
                    result.count = count;
                    table[required] = result;
                }


            }

            public RollingResult GetResult()
            {
                int v = random.Next(1, 1001);
                foreach (int key in this.table.Keys)
                {
                    if (v <= key)
                        return table[key];
                }

                return null;
            }

            public static RollingTable GetRollingTable(string name)
            {
                string path = Miner.Localization.GetPath("rolling");
                RollingTable[] tables = Miner.Localization.GetAllItems<RollingTable>(path);
                foreach(RollingTable table in tables)
                {
                     if (table.name.ToLower() == name.ToLower())
                    {
                        table.Init();

                        return table;
                    }            
                }

                return null;

            }
        }
    ```


1. Now we are going to load a rolling table to each type of the item.  We need to add a new property for the item, let’s call it “table”
    ```
    id=2008|typename=eventcoin|cap=10|desc=锁韭菜的仓 再画个大饼 10倍几率割韭菜 （直接使用 消耗10个空气币）|symbol=🥕|shortname=三爷令牌|alias=three|lifetime=1440|stat=10|table=eventone
    id=2009|typename=eventcoin|cap=10|desc=先洗脑 再割韭菜 几率10倍 收获加倍（直接使用 消耗20个空气币）|symbol=🎂|shortname=小赖老师教材|alias=study|lifetime=1440|stat=20|table=eventtwo
    id=2010|typename=eventcoin|cap=10|desc=2017年最热门游戏 100% 割韭菜 只割大的 （直接使用 消耗30个空气币）|symbol=🍴|shortname=分叉大法|alias=fork|lifetime=1440|stat=30|table=eventthree
    id=2011|typename=eventcoin|cap=10|desc=提供网上钱包服务 客户的就是你的 别客气 （直接使用 消耗50个空气币）|symbol=💼|shortname=网上钱包|alias=mybitcoins|lifetime=1440|stat=50|table=eventfour
    ```
1. This property does not exist in config, let’s override the config
    ```
    public class ItemEventConfig : ShopItemConfig
    {
       public string table;
    }
    ```
    
    In ItemEventCoin, let’s override the config, to load the table
    ```
        public override void SetConfig(ShopItemConfig config)
        {
            base.SetConfig(config);
            SwitchConfig<ItemEventConfig>();
            string tablename = ((ItemEventConfig)this.config).table;
            if (tablename != null)
                this.table = RollingTable.GetRollingTable(tablename);
        }
     ```

1. Now we can use the table to get the reward
    ```
    public override string UseItem(Miner.Miner master, string user, string target)
    {
        if (table == null)
          return null;

        uint shitcoinid = 1002;
        MinerStat stat = master.GetStat(user);
        if (stat == null)
            return null;

        //make sure we have enough shit coins, we don't need check this coin itself, as api checks for you
        if (stat.GetItemCount(shitcoinid) < this.GetStat())
            return "You don't have enough coins";

        stat.ChangeAmount(shitcoinid, -this.GetStat(), true);
        stat.ChangeAmount(this.ID(), -1, true);

        RollingResult res = this.table.GetResult();
        string sMessage = null;
        if (res == null)
        {
            sMessage = Localization.locals.IDS_SHITCOIN0;
        }
        else
        {
            stat.ChangeAmount(res.item, res.count, true);
            ShopItemBase item = master.GetInventory().GetItem(res.item.ToString());
            sMessage =  string.Format("You got {0}{1} x {2}", item.GetSymbol(), item.GetShortDesc(), res.count);

        }


        master.WriteBalance(); // make sure save at the end
        return sMessage;

    }
    ```


1. Deployment
    1. The dll need to be dropped in to the release, in the test environment, it’s auto copied
    2. The config files changed here are: shopitems.txt and rolling.txt.  these files need to be copied into the mining folder
    Also in the mining folder, give yourself the new items and a ton of shitcoins to test them

1. Add the events in specialblock.txt
    ```
    block=3707|items=2008,1,1002,5|blockdesc=仔细挖挖
    block=3708|items=2008,1,1002,5|blockdesc=仔细挖挖
    block=3709|items=2008,1,1002,5|blockdesc=仔细挖挖
    block=3710|items=2009,1,1002,15|blockdesc=仔细挖挖
    block=3711|items=2009,1,1002,15|blockdesc=仔细挖挖
    block=3712|items=2009,1,1002,15|blockdesc=仔细挖挖
    block=3713|items=2010,1,1002,25|blockdesc=仔细挖挖
    block=3714|items=2010,1,1002,25|blockdesc=仔细挖挖
    block=3715|items=2010,1,1002,25|blockdesc=仔细挖挖
    block=3716|items=2011,1,1002,45|blockdesc=仔细挖挖
    ```

Last,
If any of the api function doesn’t exist, go download the latest api  (Ref Folder in Github)
