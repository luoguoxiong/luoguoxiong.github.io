# 享元模式

如果系统中因为创建了大量类似的对象而导致内存占用过高，享元模式就非常有用了。在
JavaScript 中，浏览器特别是移动端的浏览器分配的内存并不算多，如何节省内存就成了一件非
常有意义的事情。

## 认识享元模式
假设有个内衣工厂，目前的产品有 50 种男式内衣和 50 种女士内衣，为了推销产品，工厂决
定生产一些塑料模特来穿上他们的内衣拍成广告照片。 正常情况下需要 50 个男模特和 50 个女
模特，然后让他们每人分别穿上一件内衣来拍照。不使用享元模式的情况下，在程序里也许会这
样写：
```js
var Model = function( sex, underwear){ 
 this.sex = sex; 
 this.underwear= underwear; 
}; 
Model.prototype.takePhoto = function(){ 
 console.log( 'sex= ' + this.sex + ' underwear=' + this.underwear); 
}; 
for ( var i = 1; i <= 50; i++ ){ 
 var maleModel = new Model( 'male', 'underwear' + i ); 
 maleModel.takePhoto(); 
}; 
for ( var j = 1; j <= 50; j++ ){
  var femaleModel= new Model( 'female', 'underwear' + j ); 
  femaleModel.takePhoto(); 
};
```
要得到一张照片，每次都需要传入 sex 和 underwear 参数，如上所述，现在一共有 50 种男内
衣和 50 种女内衣，所以一共会产生 100 个对象。如果将来生产了 10000 种内衣，那这个程序可
能会因为存在如此多的对象已经提前崩溃。
下面我们来考虑一下如何优化这个场景。虽然有 100 种内衣，但很显然并不需要 50 个男
模特和 50 个女模特。其实男模特和女模特各自有一个就足够了，他们可以分别穿上不同的内
衣来拍照。
现在来改写一下代码，既然只需要区别男女模特，那我们先把 underwear 参数从构造函数中
移除，构造函数只接收 sex 参数：

```js
var Model = function( sex ){ 
 this.sex = sex; 
}; 
Model.prototype.takePhoto = function(){ 
 console.log( 'sex= ' + this.sex + ' underwear=' + this.underwear); 
};
var maleModel = new Model( 'male' ), 
 femaleModel = new Model( 'female' );
for ( var i = 1; i <= 50; i++ ){ 
 maleModel.underwear = 'underwear' + i; 
 maleModel.takePhoto(); 
};
for ( var j = 1; j <= 50; j++ ){ 
 femaleModel.underwear = 'underwear' + j; 
 femaleModel.takePhoto(); 
};
```
## 内部状态和外部状态
1. 内部状态存储于对象内部。
2. 内部状态可以被一些对象共享。 
3. 内部状态独立于具体的场景，通常不会改变。
4. 外部状态取决于具体的场景，并根据场景而变化，外部状态不能被共享。

## 享元模式使用场景
享元模式要求将对象的属性划分为内部状态与外部状态（状态在这里通常指属性）。享元模式的目标是尽量减少共享对象的数量。



## 享元模式通常在以下情况下使用：

1. 大量相似对象：当应用程序需要创建大量相似的对象时，使用享元模式可以显著减少内存占用，因为它允许多个对象共享相同的状态。

2. 内存优化：如果你的应用程序受限于内存资源，享元模式可以帮助你最大程度地减少对象的内存占用，因为共享的部分不会被重复存储。

3. 性能优化：通过减少对象的数量，享元模式可以提高应用程序的性能，因为创建和销毁对象的开销通常较大。

4. 不可变对象：享元模式通常与不可变对象一起使用，以确保共享的状态不会被修改，从而避免潜在的副作用。

一些常见的使用场景包括：

1. 文本编辑器：在文本编辑器中，大部分文本都使用相同的字体、大小和颜色，只有少数文本需要不同的样式。通过共享默认样式对象，可以显著减少内存占用。

2. 游戏开发：在游戏中，例如棋盘游戏，棋子可以是享元对象，它们具有共享的属性（如颜色或形状），而位置信息是外部状态。

3. 图形和图像处理：在图形处理应用程序中，图形元素的颜色、纹理等属性可以共享，而位置和旋转信息可以是外部状态。

4. 缓存管理：在缓存管理中，可以使用享元模式来共享已经加载的资源，如图像、字体或数据库连接，以避免重复加载资源。

5. 操作系统内存管理：操作系统中的虚拟内存管理中，页面（Page）可以被看作是享元对象，多个进程可以共享相同的页面，以减少物理内存的使用。

总之，享元模式适用于需要大量对象共享相同状态的情况，以提高内存使用效率和性能。但在使用时需要注意管理共享对象的复杂性，确保外部状态传递正确。


## 图形Demo
```ts
interface Shape {
  draw(color: string, x: number, y: number): void;
}

class Circle implements Shape {
  draw(color: string, x: number, y: number): void {
    console.log(`Drawing a ${color} circle at (${x}, ${y})`);
  }
}

class Rectangle implements Shape {
  draw(color: string, x: number, y: number): void {
    console.log(`Drawing a ${color} rectangle at (${x}, ${y})`);
  }
}
class ShapeFactory {
  private shapes: { [key: string]: Shape } = {};

  getShape(shapeType: string): Shape {
    if (!this.shapes[shapeType]) {
      // 如果图形元素不存在，则创建并存储在池中
      switch (shapeType) {
        case "circle":
          this.shapes[shapeType] = new Circle();
          break;
        case "rectangle":
          this.shapes[shapeType] = new Rectangle();
          break;
        default:
          throw new Error(`Unknown shape type: ${shapeType}`);
      }
    }
    return this.shapes[shapeType];
  }
}

const factory = new ShapeFactory();

// 绘制多个红色圆形
for (let i = 0; i < 5; i++) {
  const shape = factory.getShape("circle");
  shape.draw("red", i * 20, 30);
}

// 绘制绿色矩形
const shape = factory.getShape("rectangle");
shape.draw("green", 50, 50);
```