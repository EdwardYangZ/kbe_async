# 在kbengine中实现Promise风格的Async异步同步化编程
----------
## 简介
- Async装饰器+Promise类，通过封装回调函数，轻松实现异步调用的同步化，大大简化异步逻辑处理
- 对kbengine的实体Entity类进行拓展，实现常用异步调用函数
- 远程调用会将参数进行串行化, 牺牲了部分效率
- kbengine版本 1.x - 2.0
 
----------
## 导入
1. 拷贝文件到对应目录
- scripts/common/Promise.py
- scripts/common/Async.py
- scripts/common/kbe.py
- scripts/entity_defs/interfaces/EntityBase.def
- scripts/entity_defs/interfaces/EntityCell.def
2. entity_defs中对Entity的Implements加入EntityBase或者EntityCell
````
<Implements>
    <!-- 包含Base部分就需要 EntityBase -->
	<Interface>	EntityBase		</Interface>
    <!-- 包含Cell部分就需要 EntityCell -->
    <Interface>	EntityCell		</Interface>
</Implements>
````
3. base目录中对应Entity类
````
import Async, kbe
class Account(KBEngine.Proxy, kbe.Base):
    def __init__(self):
		KBEngine.Proxy.__init__(self)
        
    @Async.async_func
    def foo():
        pass
````
4. cell目录中对应Entity类
````
import Async, kbe
class Player(KBEngine.Entity, kbe.Entity):
    def __init__(self):
		KBEngine.Entity.__init__(self)
        
    @Async.async_func
    def foo():
        pass
````
----------
## 基本用法

### Promise:
Promise的概念来自于javascript, 为解决异步问题而生，这里的Promise经过简化之后：
- 创建对象之后只能由'未resolve'状态->'已resolve'，即'resolve事件'
- 通过.resolve传入result数据，如果'已resolve'，则调用不超过
- 通过.then绑定'resolve事件'触发的回调，可绑定多个，如果已经处于'已resolve'状态，立刻触发回调
- 可在创建promise对象时把resolve的调用逻辑确定
````
promise = Promise()
# 创建promise

promise.then(print)
# 绑定resolve回调, resolve之后打印出结果
 
promise.resolve('end')
# >>> end

promise.then(lambda r: print(r + r))
# 之后的then会立马调用回调函数
# >>> endend

def _func(resolve):
    addCallback(resolve)
Promise(_func)
# 在创建时绑定另一个回调
````

### Async装饰器:
````
import Async, time, Promise

def foo():
    def _fn(resolve):
        resolve('foo end')
    return Promise.Promise(_fn)

@Async.async_func
def func():
    res = yield foo()
    return ret + ' func end'

# 被装饰的函数中可以通过 yield + promise 来挂起函数执行, promise resolve 之后会把结果变量放入res, 函数接着执行

f = func()
# 被Async.async_func装饰的函数调用会返回一个promise对象, 被装饰函数执行到return则是调用promise.resolve(..)
 
f.then(print)
# >>> foo end func end
# 打印被装饰函数的return结果

@Async.async_func
def func2():
    res = yield func()
# 同样的, 异步函数中也可以yield另一个异步函数
````
----------
## kbe拓展:
kbengine的内置函数都是通过回调实现异步逻辑, 用promise对回调进行封装就可以改造为异步调用函数
 
- kbe.Entity.delay:
    一次性定时器
- kbe.Entity.onRequest:
    远程请求接收端, 需要**注册远程函数**
- kbe.Entity.onResponse:
    远程请求回应端, 需要**注册远程函数**
- kbe.Entity.request:
    远程请求发起端, 传入远程对象, 远程调用函数名称, 调用参数来发起远程请求, 配合onRequest/onResponse实现远程参数和回应数据传递
- kbe.Base.whenGetCell:
    base中的entity, 等待其cell创建完毕, 也就是 onGetCell 回调函数被调用时, 用于异步获取base的cell对象
- kbe.Base.whenLoseCell:
    base中的entity, 等待其cell移除, 也就是 onLoseCell 回调函数被调用时
 
----------
## 用例1

* kbe原生异步处理方式, 一般用中间变量来等待接下来的处理, 例: Player 进入 Room
```
# base/Player.py
class Player(KBEngine.Proxy):
    def __init__(self):
        KBEngine.Proxy.__init__(self)

    def enterRoom(self, roomCell):
        ''' 进入房间, 在 Player.def 中声明为远程调用 '''
        self.createCellEntity(roomCell)

# base/Room.py
class Room(KBEngine.Base):
    def __init__(self):
        KBEngine.Base.__init__(self)
        self._enterPlayers = []
        self.createInNewSpace(0)

    def onGetCell(self):
        ''' cell 创建完成 '''
        if len(self._enterPlayers):
            for p in self._enterPlayers:
                self.doPlayerEnter(p)
            self._enterPlayers = []
    
    def doPlayerEnter(self, player):
        ''' 玩家进入房间, 在 Room.def 中声明为远程调用 '''
        if self.cell:
            player.enterRoom(self.cell)
            return
        self._enterPlayers.append(player)
```

* 异步同步化处理, 去掉所有由异步调用引起的中间变量, 把 Room 改造一下
```
# Room.def 中增加接口 EntityBase

# base/Room.py
class Room(KBEngine.Base, kbe.Base):
    def __init__(self):
        KBEngine.Base.__init__(self)
        self.createInNewSpace(0)

    @Async.async_func
    def doPlayerEnter(self, player):
        ''' 玩家进入房间, 在 Room.def 中声明为远程调用 '''
        if not self.cell:
            # 等到 cell 创建完成
            yield self.whenGetCell()
        player.createRoom(self.cell)
```

* kbe.Entity.request, 远程请求与回应的同步化, 向另一个可能为远程实体的对象发起请求,请求就是其python类中定义的函数, 最后 return 的变量会返回给发起者
```
# base/Room.py
class Room(KBEngine.Base, kbe.Base):
    def __init__(self):
        KBEngine.Base.__init__(self)
        self._players = []
        self.createInNewSpace(0)

    @Async.async_func
    def doPlayerEnter(self, player):
        ''' 玩家进入房间, 可以不用声明远程调用 '''
        if len(self._players) >= 3:
            # 房间已满
            return False
        for p in self._players:
            # 已经加入
            if p.id == player.id:
                return False
        if not self.cell:
            # 等到 cell 创建完成
            yield self.whenGetCell()
        self._players.append(player)
        player.createRoom(self.cell)
        return True

# base/Hall.py
class Hall(KBEngine.Base, kbe.Base):
    def __init__(self):
        KBEngine.Base.__init__(self)
        KBEngine.globalData['Hall'] = self
        self._rooms = []
    
    @Async.async_func
    def matchRoom(self, player):
        ''' 为玩家匹配一个房间加入, 失败就试下一个, 没有房间了就创建一个新的 '''
        for r in self._rooms:
            ret = yield self.request(r, 'doPlayerEnter', player)
            if ret:
                return True
        room = yield kbe.createBaseAnyWhere('Room', {})
        self._rooms.append(room)
        ret = yield self.request(room, 'doPlayerEnter', player)
        if not ret:
            ret = yield self.mathRoom(player)
            return ret
```

````
### 组织异步逻辑流程:
````
@Async.async_func
def startGame(self):
  '''开始对局，直对局结束'''
  self.reset()
  self.started = 1
  
  # 发牌并暂停2.5秒
  self.doFapai()
  yield self.delay(2.5)
  
  # 开始叫地主直到结束叫地主状态
  ret = yield self.doBankerJiao()
  if not ret:
    # 叫地主失败则重新开始
    self.doStart()
    return False
    
  # 开始抢地主直到抢地主结束
  bankerSeat = yield self.doBankerQiang()
  
  # 确定地主并暂停1.5秒
  self._doSetBanker(bankerSeat)
  yield self.delay(1.5)
  
  # 开始出牌直到分出胜负
  winSeat = yield self.doDiscard()
  
  # 结算
  self.doSettle(winSeat)
  yield self.delay(4.5)
  self.started = 6
  return True
````
 
