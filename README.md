# Root_Game 开发日志

## 第一阶段开发过程
  
### 01_美术资源

**场景搭建和模型设计**

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/7e74f0d2-d9e4-4c71-aa91-7f492cc153fc)
  
在游戏开发的初期，我们先进行了游戏美术的设计。主要分为两部分，一个是场景，我们使用美术资源在untiy搭建了大楼的走廊场景和各个房间的场景，二是角色模型，我们使用nomad制作了怪兽和boss的模型。


**游戏场景**

我们的游戏环境由由系统文件夹表示的各种“房间”组成，玩家可以在其中导航。每个房间都可以包含病毒（怪物）、电池（生命值增强器）或技能。

![image](https://github.com/YingTao-22019022/Final-Project-Creative-Making-Advanced-Visualisation-and-Computational-Environments-main/blob/master/photo/clip_image004.png)

**游戏中的房间**

使用 Blender、Nomad 和 C4D，我们设计了怪物来直观地代表不同类型的错误或病毒。首领怪物“超级病毒”被设计得特别具有挑战性。

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/d66e4de4-faed-4e66-825a-34240d6d29c7)

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/d7555a94-7856-4811-a6cc-aaab81a53360)


**该游戏具有五种独特技能：**

电池、JavaScript、Java、C# 和 Python。这些技能以编程语言的形式表现出来，可以用来杀死怪物或削弱boss。

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/d6f50d65-9e71-4fda-8753-0e072528f8ec)


**unity渲染管线优化**

unity的的渲染管线分为三种，默认管线，URP，HDRP。我们希望游戏有更好的画面效果，但是HDRP会占用非常大的内存，于是最后选择了URP管线。由于我们使用的部分素材是默认管线的，所以我们需要在项目中升级所有资源的材质。但是在这个过程中，我们遇到了资源损坏的问题，需要注意的是unity中的管线升级是不可逆的！必须备份项目后再进行升级。

reference：https://blog.unity.com/engine-platform/migrating-built-in-shaders-to-the-universal-render-pipeline

### 02_AI敌人

我们需要在游戏给所有敌人添加的功能有：
  - 默认状况下在特定范围巡逻
  - 检测玩家角色并自动攻击，
  - 追踪玩家角色

首先，我们通过unity的AI—导航—烘焙功能对整个场景进行烘焙，这个步骤在游戏过程中避免穿模，使障碍物无法被跨越。然后，我们需要在敌人上加入Nav Mesh Agent的组件，并根据角色模型大小设置代理导航对应的数值。接着，我们创建两个空对象startPoint和endPoint指引敌人来回地巡逻。
```C#
private NavMeshAgent agent;
public Transform startPoint;
public Transform endPoint;
```
最初的开发阶段中，我们在update（）中使用if语句判断是否玩家靠近，当玩家靠近时自动开始攻击。然而这样做却带来了问题，有时候玩家已经靠近敌人了，敌人却一边在巡逻一边攻击玩家，没有及时追踪玩家。经过研究发现，这是因为使用的if语句无法保证当前模式的唯一性，导致同时触发巡逻模式和攻击模式，最终我们决定将敌人的状态分为三种，往前走的巡逻模式、往后走的巡逻模式和攻击模式，使用switch()语句对敌人的控制代码进行了优化，并解决了问题。

```C#
    // EnemyControl.cs
    public enum enemyState{
        toEnd,
        toStart,
        attack
    }
    
    void Update()
    {
        switch (state)
        {
            case enemyState.toEnd:
                ToEnd();
                break;
            case enemyState.toStart:
                ToStart();
                break;
            case enemyState.attack:
                Attack();
                break;
        }
    }
    
    void ToEnd()
    {
        agent.SetDestination(endPoint.position);
        float dis = Vector3.Distance(transform.position, endPoint.position);
        if(dis < 1){
            state = enemyState.toStart;
        }
        detectPlayer();
    }

    void ToStart()
    {
        // .... mostly the same as ToEnd()
    }
    
    void Attack()
    {
        agent.SetDestination(player.transform.position);

        curTime = Time.time;
        //60s后复活
        if (curTime - lastTime >= fireInterval){
            gameObject.SetActive(true);
            state = enemyState.toStart;
            //释放攻击特效
            Fire();
            lastTime = Time.time;
        }
        //如果追到玩家，释放爆炸特效，扣一滴血
        float disToPlayer = Vector3.Distance(transform.position, player.transform.position);
        if(disToPlayer <= 0.2){
            Destroy(this);
            GameObject effect = Instantiate(deadEffect, transform);
            Destroy(effect, 2f);
            Data.Instance.HP -= 1;
        }
    }
```

### 03_场景跳转管理

在游戏中，走廊进入的每一个房间都是随机的。我们制作了五个房间的场景，通过传送门的方式进行不同场景的跳转。这样的做法也避免了将大量美术资源放入一个场景而导致的卡顿现象。

**通过自动挂载脚本组件随机房间跳转**

Data.cs只会在游戏刚开始运行时调用registerRooms()，以数组的形式储存每个随机生成的房间类型，然后这个函数给每一个传送门随机的挂上加载不同类型房间的代码.
```C#
// Data.cs
private void registerRooms(){
  Debug.Log("首次加载房间");
  string[] roomName = { "room1", "room2", "room3" };

  GameObject portals = GameObject.FindGameObjectWithTag("Portal");

  int nameType = 0;

  for (int i = 0; i < 16; i++)
  {
      //以数组的形式储存了每个房间的类型
      int type = PortalIndex[i];
      if (type == 1)
      {
      //空房间
          portals.transform.GetChild(i).gameObject.AddComponent<ToToilet>();
      }
      else if (type == 2)
      {
      //有怪兽的房间
          portals.transform.GetChild(i).gameObject.AddComponent<ToAttack>();
      }
      else if (type == 3)
      {
      //Boss房间
          portals.transform.GetChild(i).gameObject.AddComponent<ToBoss>();
      }
      else
      {
      //有物品掉落的房间
          string name = roomName[RandomRoomIndex[nameType]];
          portals.transform.GetChild(i).gameObject.AddComponent<ToRoom>().roomId = name;
          nameType++;
      }
  }
  ```
  
**场景刷新后的游戏数据储存**

当实现场景跳转的功能后，我们发现了新的问题：场景跳转后游戏中的数据会被刷新，无法保持记录当前血量和道具的数量。在调研后，我们找到了解决方案：创建一个空对象Data，使他不会随场景切换消失，并将所有需要记录的游戏数据储存在里面。

```C#
void Start() {
    DontDestroyOnLoad(this);
}
```  

同时我们也利用Data.cs储存了玩家进入房间前的位置，解决了每次返回走廊时位置在起点的问题，保证了游戏的连贯性。
  
```C#
//PlayerPosReset.cs 挂载在Player对象上
public class PlayerPosReset : ManagerBase<PlayerPosReset>
{
  void Start()
      //保证第一次打开游戏时在原点
      if (Data.Instance.lastPositionBeforeGoToRoom.x != 1)
      {
          transform.position = Data.Instance.lastPositionBeforeGoToRoom;
          Debug.Log("player update to" + transform.position);
      }
  }

  public void SaveData()
  {
      Vector3 newPos;

      Vector3 playerPos = transform.position;
      Debug.Log("player pos  :" + playerPos);

      newPos = playerPos;

      //返回的时候位置在走廊中间
      if (playerPos.x < -37) newPos.x = -40;
      else if (playerPos.x > -37) newPos.x = -34;
      newPos.y = playerPos.y;
      newPos.z = playerPos.z;

      Data.Instance.lastPositionBeforeGoToRoom = newPos;
  }
}
```
加载新场景前，调用```PlayerPosReset.Instance.SaveData();```保存当前位置。

**玩家操作系统**

当玩家使用技能时，游戏会调用“子弹预制件”来创建实例化的子弹，并且还减少了UI技能计数的显示。在开发初期我使用“if”语句判断当前玩家的行为状态，然而“if”语句无法保证玩家状态的唯一性，经常同时触发多个状态，在使用“switch”语句进行优化后修复了上述问题。
![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/9cdc8c40-3941-4ec6-8ecd-b7c9200d94fe)

过程中遇到了子弹发射位置的问题。发射点是子弹实例化和发射的点。我们必须精确定位它，使子弹看起来像是来自角色的枪。为了解决这个问题，我们使用“firePoint”变量来定义子弹实例化的确切点。

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/c7ab834e-9928-4dda-af3b-2114dc0e3dea)
![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/2c53897e-5463-45d3-ba48-b97c5faf6dbd)

最后，对撞机的定位和尺寸也是一个挑战。除了玩家和怪物的技能之外，boss还有三种不同的技能。一种技能是从天而降的流星，一种技能是大范围效果技能，还有一种技能是光束。每个碰撞器都需要准确设置。我们必须确保它们的位置和大小正确，以便攻击触发合理且现实。为此，我们使用了 Unity 的碰撞器组件，并在 Unity 编辑器中调整了它们的大小和位置，直到达到预期的结果。

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/287e265d-92f8-4814-b4d5-d835694180c0)


## 第二阶段迭代过程
  
### 优化玩家角色视角和移动的控制方式

在初始版本中，我们采用第三人称视角，上下键控制前后移动，左右键控制左转和右转。经过测试后我们发现玩家左右移动不够灵敏，并且由于我们的场景比较小，第三人称的视角遮挡了了玩家视线，所以我们决定将第三人称视角改为第一人称视角，增加游戏的沉浸感。

**增加摇杆控制视角转动**

我们加入了joystick控制角色移动，摇杆可以单独控制视角转动，不会和上下左右键冲突。改动之后，玩家前后左右的移动控制和左右转动能够被同时控制。

  ![1685811541485](https://github.com/tomoko-tiba/Root_Game/assets/41440180/933aec1b-6211-4794-a2fd-9de7b48ccf3d)
  
```C++
  //arduino

  Serial.print(analogRead(0));    //Read the position of the joysticks X axis and print it on the serial port.  

  if(analogRead(0)>700){
    //左摇
    Keyboard.write('Q'); 
  }

  if(analogRead(0)<300){
    //右摇
    Keyboard.write('E'); 
  }
```

### Boss决战环节改进

在原来的版本中，玩家使用收集到的四个技能物品进行攻击，但是四个技能的攻击区别只体现在boss四个血条的变化上，缺少游戏的趣味性，互动体验不好。于是我们改进了Boss的模型，将每个技能攻击的目标对应为boss上四个部分（左手，右手，盾，后背的武器）。当其中一个血条为0时，对应的boss身体部分也会掉落，给用户更多的反馈。

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/2a514495-b6a6-4e54-8e70-b816734ab980)
![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/d9383bc0-698b-4dfd-b545-b64c82593bff)

### 反思和今后的工作

在之前的尝试中，我们的代码不够简洁；我们经常使用“if”或“switch”语句，这不仅导致许多代码不断更新，而且还导致代码冗余，导致游戏卡顿。这些实际上可以用“字典”替换以简化代码。展望未来，我们计划提高编码技能以优化游戏。

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/75dfa9bc-f5cc-445d-8e34-a238b7fad08e)


