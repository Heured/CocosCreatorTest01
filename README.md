## 遇到问题

1. 当左右方向切换过快时，角色会突然停止

```js
//源于player_ctrl.js
onKeyReleased: function (event) {
    let keyCode = event.keyCode;
    switch(keyCode) {
        case cc.macro.KEY.a:
        case cc.macro.KEY.left:
        case cc.macro.KEY.d:
        case cc.macro.KEY.right:
            this.directionX = 0;
            break;  
    }
},
```

分析：是按下相反方向键快于松开方向键导致this.directionX先=1或-1又变成0。

解决办法: 松开按键时判断是否是原来的x轴方向

```js
    onKeyReleased: function (event) {
        let keyCode = event.keyCode;
        switch(keyCode) {
            case cc.macro.KEY.a:
            case cc.macro.KEY.left:
                if(this.directionX == -1)
                    this.directionX = 0;
                break;
            case cc.macro.KEY.d:
            case cc.macro.KEY.right:
                if(this.directionX == 1)
                    this.directionX = 0;
                break;  
        }
    },
```

这样就不会发生如上情况了。

2. 在奔跑状态下触发攀爬状态时虽然动画已经改变，但是x轴速度没有归零

解决办法: 在_onclimb函数里添加初始化速度。

```js
//源于player_ctrl.js
//攀爬动作
    _onClimb: function(){
        // if(this.climbing)
        //     return;
        this.climbing = true;
        this._armatureDisplay.playAnimation('Climb',0);

        //初始化x轴速度
        this.directionX = 0;
        this.rbody.linearVelocity = cc.v2(0,0);
        //关闭重力
        this.rbody.gravityScale = 0;
        //进入传感器模式
        // this.phy_collider.sensor = true;

    },
```

3. 在攀爬状态下，松开w(上方向键)一段时间后再按s(下方向键)，角色只会在原本位置逗留，而不会往下掉。

 ```js
//源于player_ctrl.js
_offClimb: function(){
        // if(this.climbing)
            // return;
        this.climbing = false;
        this._armatureDisplay.playAnimation('Idle',0);

        
        //打开重力
        this.rbody.gravityScale = 5;
        //退出传感器模式
        // this.phy_collider.sensor = false;
    },
 ```



分析：一段时间后，角色的刚体进入睡眠状态，重力不产生作用，所以不会往下掉。

解决办法: 在按下s(下方向键)后，给角色的y轴一个初始速度后恢复正常；或者设置不允许进入睡眠

```js
_offClimb: function(){
        // if(this.climbing)
            // return;
        this.climbing = false;
        this._armatureDisplay.playAnimation('Idle',0);

        //初始化速度
        this.rbody.linearVelocity = cc.v2(0,1);
        //打开重力
        this.rbody.gravityScale = 5;
        //退出传感器模式
        // this.phy_collider.sensor = false;
    },
```

4. 爬梯子问题：触碰到梯子时，进入可爬状态，并指定碰到的梯子为攀爬对象，按下w或 ^ ，角色进入攀爬状态，此时可以穿过平台刚体，当角色离开梯子后进入一般状态，此时不会穿过平台刚体。

   (1) 攀爬状态时可以穿过平台

   分析：可以使用物理碰撞体的sensor属性来实现，调用物理碰撞体的apply方法刷新碰撞体。(注意：调用apply会销毁然后重新生成物理碰撞体，也就意味着销毁时会与梯子触发一次碰撞结束，生成后会与梯子重新发生碰撞。)

```js
//源于player_ctrl.js
this.phy_collider = this.getComponent(cc.PhysicsBoxCollider);
this.phy_collider.sensor = true;
this.phy_collider.apply();
```

​	(2) 攀爬状态时，爬到梯子顶后进入一般状态，可以踩踏平台。

​	分析：方法跟(1)相同，只不过如果刚碰撞结束就调用状态更换也许会和梯子又一次发生碰撞，导致爬到平台上以后又进入攀爬状态。所以使用scheduleOnce方法在攀爬时离开梯子后保持攀爬状态再向上移动0.1秒再进行状态转换，这样就能保证在攀爬上平台后不进入攀爬状态。

```js
//源于player_ctrl.js
this.scheduleOnce(function(){
	this._offClimb();
},0.1);
```

5. 遇到一个小bug，平台边缘往外跳，顶到其他平台刚体后再回到原来平台，可能会导致速度不置零。

   

   分析：添加一个阈值使ad(< >)键松开后x轴速度小于这个阈值时自动置零。

```js
//源于player_ctrl.js
let v = this.rbody.linearVelocity;
if(!this.directionX&& Math.abs(v.x)<25)
    v.x = 0;
this.rbody.linearVelocity = v;
```

6. 在制作场景跳转以及用常驻节点保存数据与转场逻辑时，发现一个问题。这里用global_manager作为常驻节点，原本是采用global_manager与主菜单场景一起载入游戏的方式来记录游戏数据，但是转换到其他场景后再回到主菜单后，主菜单的按钮全失效了。

   分析：在载入主菜单场景后，scene内已经存在一个常驻节点global_manager，此时如果再载入主菜单场景，这就会使scene导入另一个global_manager，由此产生了冲突。所以解决办法也很简单，写一个逻辑判断scene内是否已经存在一个global_manager，如果不存在则导入global_manager的Prefab作为常驻节点，否则不导入。这里要注意的是，因为在场景加载时没有global_manager一同加载，所以按钮的事件绑定需要在脚本中实现。

   解决办法:

```js
//源于mainMenu.js
onLoad () {
        if(!cc.find('global_manager')){
            let scene = cc.director.getScene();
            let global_manager = cc.instantiate(this.global_manager_prefab);
            global_manager.parent = scene;
        }
        this.global_manager = cc.find('global_manager');
        let global_manager_script = 'global_manager';

    	// 绑定开始游戏按钮的点击事件
        let eventHandler_roundStart = new cc.Component.EventHandler();
        eventHandler_roundStart.target = this.global_manager;
        eventHandler_roundStart.component = global_manager_script;
        eventHandler_roundStart.handler = 'round_start';
        this.round_start_btn.getComponent(cc.Button).clickEvents.push(eventHandler_roundStart);

    	// 绑定退出游戏按钮的点击事件
        let eventHandler_gameEnd = new cc.Component.EventHandler();
        eventHandler_gameEnd.target = this.global_manager;
        eventHandler_gameEnd.component = global_manager_script;
        eventHandler_gameEnd.handler = 'game_end';
        this.game_end_btn.getComponent(cc.Button).clickEvents.push(eventHandler_gameEnd);
    },
```

7. 发现一个有意思的bug，按w -> 落地瞬间同时按 a/d+w -> 最高点碰到平台类时连按w。可能触发二段跳，跳到平时跳不到的高度的平台上，不决定修复这个bug，留着做彩蛋吧。