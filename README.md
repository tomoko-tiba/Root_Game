# Root_Game
> Note: Standard Readme is designed for open source libraries. Although it's [historically](README.md#background) made for Node and npm projects, it also applies to libraries in other languages and package managers.

**Requirements:**
  - Be called README.md (with capitalization).
  - If the project supports i18n, the file

  _Note: This is only a navigation guide for the specification, and does not define or mandate terms for any specification-compliant documents._

```markdown
# Standard Readme Style _(standard-readme)_
```
  
## 第一阶段开发过程
  
### 01_美术资源

**场景搭建和模型设计**

![image](https://github.com/tomoko-tiba/Root_Game/assets/41440180/7e74f0d2-d9e4-4c71-aa91-7f492cc153fc)
  
在游戏开发的初期，我们先进行了游戏美术的设计。主要分为两部分，一个是场景，我们使用美术资源在untiy搭建了大楼的走廊场景和各个房间的场景，二是角色模型，我们使用nomad制作了怪兽和boss的模型。

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
    
    oid ToEnd()
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
        agent.SetDestination(startPoint.position);
        float dis = Vector3.Distance(transform.position, startPoint.position);
        if (dis < 1){
            state = enemyState.toEnd;
        }
        detectPlayer();
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
  
## 第二阶段迭代过程
  
  
