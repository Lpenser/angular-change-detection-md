---
## @color[#DC143C](Angular推进器)
### Angular从原理到应用 - @color[orange](变更检测)
---
### 什么是状态
- 获取程序的内部状态
- 使其以某种方式对用户界面可见
- 状态可以是: objects, Arrays, Primitives...
- Value Types: string number boolean null undefined
- Reference Types: array object function
---
### 当检测发生在运行时
- 模型中的变化 -> 更新DOM的位置
- 操作DOM开销昂贵
- <img style="background: #0c4eb2; padding: 0 1em; width: 300px;text-align: center" src="https://blog.thoughtram.io/images/cd-4.svg">
---
### 多种解决方案
- HTTP 请求 + server 重新渲染
- 区分前后DOM的差异, 渲染不同的部分
- React Virtual DOM
---
### 回到Angular
- 变化什么时候发生?
```javascript
@Component({
  template: `
    <h1>{{firstname}} {{lastname}}</h1>
    <button (click)="changeName()">Change name</button>
  `
})
export class AppComponent {

  firstname:string = 'Material';
  lastname:string = 'Angular';

  changeName() {
    this.firstname = 'Awesome';
    this.lastname = 'Angular';
  }
}
```
---
### 另一个例子
```javascript
@Component()
export class ContactsComponent implements OnInit{

  people:Person[] = [];

  constructor(private http: HttpClient) {}

  ngOnInit() {
    this.http.get('/people')
      .map(res => res.json())
      .subscribe(people => this.people = people);
  }
}
```
---
### 什么会触发变更?
- Events - click, submit...事件
- XHR - 从远端服务器获取数据
- Timers - setTimeout(), setInterval() 浏览器web api
+++
### 触发变更?

#### Asynchronous
---
### 谁通知了Angular? 
- [Zone.js](https://github.com/angular/zone.js#augmenting-a-zones-hook)
- [NgZone](https://angular.io/api/core/NgZone) implements from Zone.js
```javascript
// 源码简化
class ApplicationRef {

  changeDetectorRefs:ChangeDetectorRef[] = [];
  // applicationRef在构造器中监听onTurnDone事件
  constructor(private zone: NgZone) {
    this.zone.onTurnDone
      .subscribe(() => this.zone.run(() => this.tick());
  }
// tick函数遍历所有的探测器的接口/对象 对其执行检测
  tick() {
    this.changeDetectorRefs
      .forEach((ref) => ref.detectChanges());
  }
}
```
---
### 变更检测是如何进行的呢?
- Key: 每一个组件都有属于自己的变更检测器(change detector)
- <img style="background: #0c4eb2; padding: 0 1em; width: 300px" src="https://blog.thoughtram.io/images/cd-tree-2.svg"> <img style="background: #0c4eb2; padding: 0 1em; width: 300px" src="https://blog.thoughtram.io/images/cd-tree-7.svg">
- 变更检测树 change detector tree: 有向图 数据流从上而下
---
### 数据流单向从上到下?
- 变更检测 from top to bottom
- 优点多多
- unidirectional data flow 单向数据流
---
### 认清单向数据流
- Angular 分离更新应用程序模型并将视图中模型的状态反映为两个不同的阶段
- 开发者负责更新应用model,变更检测负责把model的状态反映到视图
- @Output()会在变更检测执行前运行
- 单向数据流的意思是指在变更检测期间属性绑定变更的架构
---
### BFS OR DFS?
- 看上去像是奇怪的BFS
- <img data-src="https://cdn-images-1.medium.com/max/1600/1*XFBDFfCa4Trq_C9M9AZiYQ.gif" src="https://cdn-images-1.medium.com/max/1600/1*XFBDFfCa4Trq_C9M9AZiYQ.gif">
---
### 实际上是DFS
- 当Angular检查当前组件时,它调用子组件上的生命周期钩子,但渲染当前组件DOM
- <img data-src="https://cdn-images-1.medium.com/max/1600/1*4i4InJWyGkLJfV0IcUsZQw.gif" src="https://cdn-images-1.medium.com/max/1600/1*4i4InJWyGkLJfV0IcUsZQw.gif">
---
### NgDoCheck钩子和变更检测
- 更新子组件的属性
- 调用位于子组件中的NgDoCheck生命周期钩子
- 更新当前组件的DOM
- 向子组件执行变更检测
---
### 对ngDoCheck的误会
- stackoverflow上常见的疑问
- @color[blue](为什么在OnPush策略下,即使组件没有属性更新,ngOnCheck钩子仍然被调用了?变更检测是怎么回事?)
---
### 和变更检测最相关的一些
- 更新所有子组件/指令的绑定属性
- 三个生命周期钩子：ngOnInit，OnChanges，ngDoCheck
- 更新当前组件的 DOM
- 为子组件执行变更检测
- 为所有子组件/指令调用当前组件的 ngAfterViewInit 生命周期钩子
- 变更检测同样会触发生命周期钩子,甚至在检查父组件时会触发子组件的钩子
---
### 检测循环内?
```
ComponentA
    ComponentB
        ComponentC
```
```
检测 A component:
  - 更新B的输入绑定
  - 执行B组件的NgDoCheck钩子
  - 更新A组件的DOM
 
 (当Input()绑定发生了变化)检测 B component:
    - 更新C的输入绑定
    - 执行C组件的NgDoCheck钩子
    - 更新B组件的DOM
 
   检测 C component:
      - 更新C组件的DOM
```
---
### ngDoCheck有什么用?
- 配合markForCheck 和 OnPush 
```javascript
export class AppComponent {
  @Input() data;

  // 初始化并用来存储之前的id
  public id;

  constructor(private cdr: ChangeDetectorRef) {}

  ngOnChanges() {
    // 当data这个object改变时,更新id
    this.id = this.data.id;
  }

  ngDoCheck() {
    // 在ngDoCheck中检测data这个object的属性是否变化
    if (this.id !== this.data.id) {
      this.cdr.markForCheck();
    }
  }
}
```
---
### 变更检测效率如何?
- 感觉上很慢实际上很快 得益于 Angular 生成 VM 友好的代码
- VM 不喜欢动态不确定的代码 VM的优化得益于 object的单态 而不是多态
- Angular在运行时 创造变更检测器 - 单态 - 确定的model
- Don't worry, Angular 帮我们处理好了这些复杂的部分
---
### 更聪明的变更检测
- Angular默认的变更检测是自动的
- 两个好帮手: Immutable(不可变) & Observables
---
### 理解可变和不可变(Mutability)
- reference 没变 但是 property 改变 -> Angular负责地进行检测
```javascript
@Component({
  template: '<child [data]="data"></child>'
})
export class ParentComponent {

  constructor() {
    this.data = {
      name: 'Button',
      email: 'github.com'
    }
  }

  changeData() {
    this.data.name = 'Tommy';
  }
}
```
---
### 不可变对象
- reference change

```javascript
var data = someAPIForImmutables.create({
              name: 'Button'
            });

var data2 = data.set('name', 'Tommy');

data === data2 // false reference are different
```
---
### 优化?
- 变更检测可以跳过某些component子树
- @Input()属性immutable
- 需要告诉angular 
---
### 比如
- OnPush Strategy
```javascript
@Component({
  template: `
    <h2>{{data.name}}</h2>
    <span>{{data.email}}</span>
  `
  // onPush 策略会在@Input()的内容属性不变时生效
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AppComponent {
  @Input() data;
}
```
---
### 结果?
- immutable object + OnPush
- <img style="background: #0c4eb2; padding: 0 1em; width: 400px" src="https://blog.thoughtram.io/images/cd-tree-8.svg">
---
### Observables
- 和immutable不同
- Observables + OnPush?
---
### 简单的购物车
- @Input() addItemStream reference
```javascript
@Component({
  template: '{{count}}',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class CartComponent implements OnInit {

  @Input() addItemStream: Observable<any>;
  count = 0;

  ngOnInit() {
    this.addItemStream.subscribe(() => {
      this.count++; // 变更出现在OnInit hook
    })
  }
}
```
---
### 怎么办?
- 全部component设置为OnPush
- <img style="background: #0c4eb2; padding: 0 1em; width: 400px" src="https://blog.thoughtram.io/images/cd-tree-10.svg">
---
### Angular不知道  但我们知道
- markForCheck from ChangeDetectorRef 

```javascript
@Component({
  template: '{{count}}',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class CartComponent implements OnInit {

  @Input() addItemStream: Observable<any>;
  count = 0;
  // 注入ChangeDetectorRef
  constructor(private cdr: ChangeDetectorRef)
  ngOnInit() {
    this.addItemStream.subscribe(() => {
      this.count++; // 变更出现在OnInit hook
      this.cdr.markForCheck(); // 人为通知angular检测这个component
    })
  }
}

```
---
### 不慌了
- Observables 事件已经被触发了(变更检测前)
- <img style="background: #0c4eb2; padding: 0 1em; width: 400px" src="https://blog.thoughtram.io/images/cd-tree-12.svg">
---
### 结果
- Observables(变更检测后)
- <img style="background: #0c4eb2; padding: 0 1em; width: 400px" src="https://blog.thoughtram.io/images/cd-tree-13.svg">
---
### 另一个应用场景
- setTimeout&setInterval 
```javascript
@Component({
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class AppComponent implements OnInit{
  data = [{name: 'Button'}];  
  constructor(public cdr: ChangeDetectorRef) {}

  ngOnInit() {
    setTimeout(() => {
      this.data.push({name: 'Tommy'});
      this.cdr.markForCheck(); // setTimeout + OnPush也需要配合 markForCheck使用
    }, 2000);
  }  
}
```
---
### 变更检测的种类
- CheckOnce(move to changeDetectorRef)
- Checked(Depreciated)
- CheckAlways(Depreciated)
- Detached(move to changeDetectorRef)
- OnPush(In using)
- Default(In using)
- 自己去探索
---
### 一个可能会踩到的坑
- Pure pipe
```javascript
{{data | CustomizedPipe}}
// data is a reference type, customized pipe may not be triggering.
```
- data的属性发生了变化 但是reference没变
---
### 解决方案
- Impure pipe
```javascript
@Pipe({
  name: 'CustomizedPipe',
  pure: false
})
```
- 类似于markForCheck
- Angular创建了一个impure的多个实例，并在每个检测周期调用它定义的转换方法
- options: 尽量使用immutable type data
---
## ExpressionChangedAfterItHasBeenCheckedError
- 检查已经传给子组件用来更新其属性的值，是否与当前将要传入的值相同
- 检查已经传给当前组件用来更新DOM的值，是否与当前将要传入的值相同
- 递归检查每一个子组件
---
## 真实场景会更复杂
- shared service
- Observables
- dynamic component initialize 动态组件初始化
---
## 解决方案
- 异步更新
- 强制进行变更检测,但是会触发子组件的变更检测,再次导致父组件属性改变
- 为什么需要这个机制: 组件树稳定
---
### 结束之前
#### Q&A