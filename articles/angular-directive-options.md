---
title: "Angular の Directive の Options をみる"
emoji: "🔍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Angular"]
published: false
---

とりあえず Angular に入門してみたものの、チュートリアルをいくつかやってみたぐらいではまだまだ分かりません。。

どうやらよく使う `Component` は `Directive` を継承したもののようなので、まずは `Directive` の `Options` をみていこうと思います。

[Angular - Directive](https://angular.io/api/core/Directive)

上記のドキュメントをなぞったような内容です。

## selector

> The CSS selector that identifies this directive in a template and triggers instantiation of the directive.

なんとなくそんな感じでしが、CSS selector そのものを指定していたんですね。

https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Selectors

- `app-example-component`
  - component の指定でよくみるこれは Type Selector
- `[app-example-directive]`
  - 属性 directive の指定で見るこれは Attribute selector

他にも柔軟に指定できます。

例えばよく使う `NgForm` は以下のような感じです。
https://github.com/angular/angular/blob/4dcbb6aef9ec6d1f1fe9a926d0b40c72139a013b/packages/forms/src/directives/ng_form.ts#L96


## inputs

> Enumerates the set of data-bound input properties for a directive

ドキュメントには、さらに以下のように記載されています。

> The inputs property defines a set of directiveProperty to bindingProperty configuration:
> 
> - `directiveProperty` specifies the component property where the value is written.
> - `bindingProperty` specifies the DOM property where the value is read from.

ドキュメントにも小さいサンプルはあるのですが、別に書いてみます。

```ts
@Directive({
  selector: '[appChangeColor]',
  standalone: true,
  inputs: ['backgroundColor', 'weight: fontWeight', 'id: sample-id']
})
export class ChangeColorDirective implements OnInit {
  backgroundColor: string = 'white';
  weight: string = 'normal';
  id: string = '';

  constructor(private el: ElementRef) { }

  ngOnInit() {
    this.el.nativeElement.style.backgroundColor = this.backgroundColor;
    this.el.nativeElement.style.fontWeight = this.weight;

    console.log('id: ' + this.id);
  }

  @Input() set appChangeColor(color: string) {
    this.el.nativeElement.style.color = color;
  }
}
```

利用側

```html
<p appChangeColor="blue" 
  [backgroundColor]="'yellow'" 
  [fontWeight]="'bold'" 
  [sample-id]="'123'">appChangeColor="blue"</p>
<p [appChangeColor]="'red'">[appChangeColor]="'red'"</p>
```

見ての通りで、`backgroundColor` は単に書き込み用の property なので単純ですが、 `weight` と `id` はエイリアスのようなものを指定する形になっています。

上記の例は、`@Input` デコレータでもほぼ同じように記述できます。

```ts
@Directive({
  selector: '[appChangeColor2]',
  standalone: true
})
export class ChangeColor2Directive implements OnInit {
  constructor(private el: ElementRef) { }

  ngOnInit() {
    this.el.nativeElement.style.backgroundColor = this.backgroundColor;
    this.el.nativeElement.style.fontWeight = this.weight;
  }

  @Input() backgroundColor: string = 'white';
  @Input('fontWeight') weight: string = 'normal';
}
```

```html
<p appChangeColor2="green" 
  [backgroundColor]="'yellow'" 
  [fontWeight]="'bold'">appChangeColor2="green"</p>
```

## outpus

> Enumerates the set of event-bound output properties.

`inputs` と使い方は同じです。

```ts
@Component({
  selector: 'app-change-color',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './change-color.component.html',
  styleUrls: ['./change-color.component.scss'],
  outputs: ['colorChange', 'fontChange: onChangeFont'],
})
export class ChangeColorComponent {
  colorChange: EventEmitter<string> = new EventEmitter();
  fontChange: EventEmitter<string> = new EventEmitter();
}
```

```html
<button (click)="colorChange.emit('red')">colorChange</button>
<button (click)="fontChange.emit('bold')">fontChange</button>
```

続いて利用側です。

```html
<p [style.color]='colorOfOutputSample' 
  [style.font-weight]='fontWeightOfOutputSample'>Outputs Sample</p>
<app-change-color (colorChange)="onColorChange($event)" 
  (onChangeFont)="onChangeFont($event)"></app-change-color>
```

```ts
export class DirectiveSampleComponent {
  colorOfOutputSample: string = 'black';
  fontWeightOfOutputSample: string = 'normal';

  onColorChange($event: string) {
    console.log($event);
    this.colorOfOutputSample = $event;
  }

  onChangeFont($event: string) {
    this.fontWeightOfOutputSample = $event;
  }
```

|ボタンクリック前|クリック後|
|---|---|
|![](https://storage.googleapis.com/zenn-user-upload/caabaada2bd3-20230409.png)|![](https://storage.googleapis.com/zenn-user-upload/84b1eef968a0-20230409.png)|

こちらも、`@Output` デコレータで同じことができます。

```ts
@Component({
  selector: 'app-change-color2',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './change-color2.component.html',
  styleUrls: ['./change-color2.component.scss']
})
export class ChangeColor2Component {
  @Output() colorChange: EventEmitter<string> = new EventEmitter();
  @Output('onChangeFont') fontChange: EventEmitter<string> = new EventEmitter();
}
```

## providers

> Configures the injector of this directive or component with a token that maps to a provider of a dependency.

`providers` は DI に関する項目のようです。

- https://angular.io/guide/glossary#injector
- https://angular.io/guide/glossary#di-token
- https://angular.io/guide/glossary#provider

以下のページがとても分かりやすいです。

https://zenn.dev/lacolaco/books/angular-after-tutorial/viewer/dependency-injection

今回は token をいくつか作成して使ってみることにします。

```ts
import {InjectionToken} from '@angular/core';

// 自身に注入されるトークンを想定
export const SAMPLE_TOKEN = new InjectionToken('sampleToken');

// 親コンポーネントにあるトークンを想定
export const PARENT_PROVIDED = new InjectionToken('ParentProvided');

// root にあるトークン
export const ROOT_PROVIDED = new InjectionToken('RootProvided',
  { providedIn: 'root', factory: () => 'root provided value' }
);
```

まずは今回の中心になる Directive です。まずは `providers` で `SAMPLE_TOKEN` を渡していることに注目してください。  
今回は、さらに比較のためにコンストラクタからも注入しています。

```ts
@Directive({
  selector: '[appProvider]',
  standalone: true,
  providers: [{provide: SAMPLE_TOKEN, useValue: 'sample value'}]
})
export class ProviderDirective implements OnInit {
  providedValue = inject(SAMPLE_TOKEN);
  // read from parent component
  parentProvidedValue = inject(PARENT_PROVIDED);

  rootProvidedValue = inject(ROOT_PROVIDED);

  // コンストラクタでも注入
  constructor(
    @Inject(SAMPLE_TOKEN) 
    private providedValueFromConstructor: string,

    // Host に限定するため取得できない
    @Host() @Optional() @Inject(PARENT_PROVIDED) 
    private parentProvidedValueFromConstructor: string,

    @Optional() @Inject(PARENT_PROVIDED) 
    private parentProvidedValueFromConstructor2: string,

    @Inject(ROOT_PROVIDED) private rootProvidedValueFromConstructor: string,
    ) { }

  ngOnInit() {
    console.log(`provided value: ${this.providedValue}`);
    console.log(`parent provided value: ${this.parentProvidedValue}`);
    console.log(`root provided value: ${this.rootProvidedValue}`);

    console.log(`provided value from constructor: ${this.providedValueFromConstructor}`);
    console.log(`parent provided value from constructor: ${this.parentProvidedValueFromConstructor}`);
    console.log(`parent provided value from constructor 2: ${this.parentProvidedValueFromConstructor2}`);
    console.log(`root provided value from constructor: ${this.rootProvidedValueFromConstructor}`);
  }
}
```

続いてこれを利用している親コンポーネントです。(注目する部分以外は省略しています)  
親側で `PARENT_PROVIDED` を注入しています。

```ts
@Component({
  selector: 'app-directive-sample',
  standalone: true,
  imports: [CommonModule, ProviderDirective],
  providers: [{provide: PARENT_PROVIDED, useValue: 'ParentProvidedValue'}]
})
```

これで実行すると、console の結果は以下のようになります。

```
provided value: sample value
parent provided value: ParentProvidedValue
root provided value: root provided value

// 以下はコンストラクタ注入
provided value from constructor: sample value
parent provided value from constructor: null // HOST に限定されたもの
parent provided value from constructor 2: ParentProvidedValue
root provided value from constructor: root provided value
```

`providers` オプションから渡されたものは、全て取得できていることがわかります。  
同様にコンストラクタからも取得できていますが、`@Host` を指定したものだけは取得できていません。

他にもいくつか注入を制御できるデコレータが存在します。(`@Host` と `@Self` の違いがよく分からない)

[Angular - Dependency injection in action](https://angular.io/guide/dependency-injection-in-action#qualify-dependency-lookup-with-parameter-decorators)

## exportAs

> Defines the name that can be used in the template to assign this directive to a variable.

`exportAs` は `NgForm` や `NgModel` でよくお世話になっている機能ですね。

https://github.com/angular/angular/blob/0f2937ef83d1b394159990cb5346a0b097eb9242/packages/forms/src/directives/ng_model.ts#L134-L139

https://github.com/angular/angular/blob/4dcbb6aef9ec6d1f1fe9a926d0b40c72139a013b/packages/forms/src/directives/ng_form.ts#L95-L102

簡単な例を自分でも書いてみます。

```ts
@Directive({
  selector: '[appExportAs]',
  standalone: true,
  exportAs: 'exportAsSample'
})
export class ExportAsDirective {
  get exportedValue() {
    return 'exported value';
  }
}
```

続いて利用側です。

```html
<p appExportAs #e="exportAsSample">export as sample {{e.exportedValue}}</p>
```

これで、`exportedValue` の内容も表示されることになります。

## queries

> Configures the queries that will be injected into the directive.

説明からはパッと分からないのですが、`ContentChild` と `ViewChild` に関する設定のようです。

簡単なサンプルを書きます。

まずは Directive を 2 種類用意します。

```ts
@Directive({
  selector: 'app-contentChild-sample',
  standalone: true
})
export class ContentChildSampleDirective {}

@Directive({
  selector: 'app-viewChild-sample',
  standalone: true
})
export class ViewChildSampleDirective {}
```

続いて、今回の主役になる Component を用意します。ContentChild を使うので `ng-content` を用意しています。

```html
<ng-content></ng-content>

<app-viewChild-sample></app-viewChild-sample>
<app-viewChild-sample></app-viewChild-sample>
```

```ts
@Component({
  selector: 'app-content-child-sample',
  standalone: true,
  imports: [CommonModule, ViewChildSampleDirective],
  templateUrl: './content-child-sample.component.html',
  styleUrls: ['./content-child-sample.component.scss'],
  queries: {
    contentChildren: new ContentChildren(ContentChildSampleDirective),
    viewChildren: new ViewChildren(ViewChildSampleDirective)
  }
})
export class ContentChildSampleComponent {

  contentChildren!: QueryList<ContentChildSampleDirective>;
  viewChildren!: QueryList<ViewChildSampleDirective>;

  ngAfterContentInit() {
    console.log(`contentChildren count: ${this.contentChildren.length}`);
  }

  ngAfterViewInit() {
    console.log(`viewChildren count: ${this.viewChildren.length}`);
  }
}
```

最後に、このコンポーネントを親側に組み込みます。(名前づけが悪くてややこしいですが、`app-content-child-sample` がコンポーネントで、`app-contentChild-sample` がディレクティブです)

```html
<app-content-child-sample>
  <app-contentChild-sample></app-contentChild-sample>
  <app-contentChild-sample></app-contentChild-sample>
  <app-contentChild-sample></app-contentChild-sample>
</app-content-child-sample>
```

この状態で動作させると、console には以下が表示されます。

```
contentChildren count: 3
viewChildren count: 2
```

ViewChild, ContentChild は以下の記事がとても参考になりました 🙏

https://www.digitalocean.com/community/tutorials/angular-viewchild-access-component-ja

https://tkzo.jp/blog/angular6-view-child-view-children/

https://tkzo.jp/blog/angular6-view-child-content-child/

## host

> Maps class properties to host element bindings for properties, attributes, and events, using a set of key-value pairs.

このディレクティブのホストエレメントに対してそのプロパティや属性、イベントに対してバインディングを書けるというものです。

ここでは、Angular でよくお目にかかるものを二つ程引用しておきます。

https://github.com/angular/angular/blob/edc3bb180fc197d1744e17d0267c3f562c99ec0d/packages/forms/src/directives/validators.ts#L356-L362

https://github.com/angular/angular/blob/4dcbb6aef9ec6d1f1fe9a926d0b40c72139a013b/packages/forms/src/directives/ng_form.ts#L95-L102

`required` の属性できちんとバリデータが動作するのが不思議だったんですが、これが働いていたんですね。

チュートリアルではよく `reset` も出てくるのですが、ここでバインドされていたんですねぇ。

## jit

現状よく分からないのでスキップします。

## standalone

> Angular directives marked as standalone do not need to be declared in an NgModule. Such directives don't depend on any "intermediate context" of an NgModule (ex. configured providers).

今回書いたサンプルも基本的には standalone で書いているのですが、私自身はこれの出自とかがよくわかっていないので、`NgModule` に対してのメリット・デメリットも理解できていません。

[Angular - Getting started with standalone components](https://angular.io/guide/standalone-components)

## hostDirectives

> Standalone directives that should be applied to the host whenever the directive is matched. By default, none of the inputs or outputs of the host directives will be available on the host, unless they are specified in the inputs or outputs properties.

https://blog.lacolaco.net/2022/10/angular-host-directives-observer-directive/

上記の記事によると、別のコンポーネントに直接合成するということができるらしい。

ちょっと自分ではサンプルが書けそうにないのでここまでに留めておきます。

## サンプルコード

今回書いたサンプルコードは以下にあります。試しながら書いていったので命名がごっちゃゴチャですが。

https://github.com/dany1468/angular_directive_study