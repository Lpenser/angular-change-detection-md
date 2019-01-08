---
## @color[#DC143C](Angular推进器)
### Angular从原理到应用 - @color[orange](变更检测)
---
### 如何触发angular变更
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
### BFS OR DFS?
- 看上去像是奇怪的BFS
- <img data-src="https://cdn-images-1.medium.com/max/1600/1*XFBDFfCa4Trq_C9M9AZiYQ.gif" src="https://camo.githubusercontent.com/3814ecb55df12cd257d544f4ec37f61d91178911/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f313630302f312a5846424446664361345472715f43394d39415a6959512e676966">
---
### 实际上是DFS
- 当Angular检查当前组件时,它调用子组件上的生命周期钩子,但渲染当前组件DOM
- <img data-src="https://cdn-images-1.medium.com/max/1600/1*4i4InJWyGkLJfV0IcUsZQw.gif" src="https://camo.githubusercontent.com/36d2d157bba9f01c23d4d2f9d54f8475dc9189fa/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f313630302f312a346934496e4a5779476b4c4a665630496355735a51772e676966">
---
### NgDoCheck钩子和变更检测
- 更新子组件的属性
- 调用位于子组件中的NgDoCheck生命周期钩子
- 更新当前组件的DOM
- 向子组件执行变更检测
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
### 结束之前
#### Q&A