---
title: "react-springで始めるお手軽アニメーション実装"
emoji: "📹️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["reactspring", "animation", "nextjs"]
published: true
published_at: 2025-03-13 09:00
publication_name: "aldagram_tech"
---

こんにちは！[アルダグラム](https://aldagram.com/about/)でエンジニアをしている今町です。

皆さん、React アプリケーションでアニメーションを実装してみようと思ったことはありますか？私自身、「いや…、アニメーションとか敷居が高そう…」と思って尻込みしていました。そんな中で、シンプルで直感的にアニメーションを実現できるライブラリを見つけたので紹介します。

## react-spring とは？

[react-spring](https://www.react-spring.dev/) は、React アプリケーションで自然で滑らかなアニメーションを実現するためのアニメーションライブラリです。主な特徴は以下です。

1. **物理ベースのアニメーション**
   バネ（スプリング）の動きを模倣することで、より自然で直感的なアニメーションを作成できます。
1. **シンプルかつ柔軟な API**
   複雑なアニメーションも簡単に実装できるように、直感的な宣言的 API が提供されています。
1. **React との統合が容易**
   React のコンポーネントや Hooks とシームレスに連携し、状態に応じたアニメーションの管理が簡単です。

このライブラリを利用することで、ユーザーインターフェースの動きをより自然で魅力的に演出できます（ホンネ: アニメーションつけるとなんかカッコイイじゃん）

## インストール

`react-dom`、`react-native`、`react-three-fiber`、`react-konva`、`react-zdog` にそれぞれ対応しています。今回は、Next.js を用いた Web アプリケーションを想定しているので、以下のようにインストールします

:::message
`react-spring` を直接インストールすることもできますが、用途の応じてパッケージに分かれているので、特定のターゲットに絞ってインストールすることをオススメします。
:::

```zsh
npm install @react-spring/web
```

## 具体例

子コンポーネントをラップするカタチで使用できるアニメーション用のコンポーネントを作ってみましょう。

### from と to で状態の変化を指定してアニメーションさせる

下から子コンポーネント（アイコン）をフェードインさせるアニメーションを作ってみましょう。
`useSpring` hooks に設定するのは以下の 2 点になります。

- `from` と `to`: アニメーション開始から終了までの状態を設定
- `config`: アニメーションの動きの速度を調整するための物理パラメータ
  - `tension`: バネの強さ（加速度みたいなもの）
  - `friction`: ブレーキの強さ（数値が大きいほど減速効果が高まります）

ここでは、transform を設定することで、50px 下から Y 軸方向にアイコンを移動させています。また、opacity を 0 → 1 に変化させることで、下からふわっと出現する感じを演出できます。
`tension` と `friction` の数値調整は、個人的な直感によるところも大きいので、try & error を繰り返して調整してみてください。

```ts:FadeInFromBottom.tsx
import { useSpring, animated } from "@react-spring/web";

type Props = {
  children: React.ReactNode;
};

/**
 * 下からフェードインするアニメーション
 */
const FadeInFromBottom = ({ children }: Props) => {
  const props = useSpring({
    from: { opacity: 0, transform: "translateY(50px)" },
    to: { opacity: 1, transform: "translateY(0px)" },
    config: { tension: 200, friction: 20 }, // アニメーションの速度や動き具合を調整
  });

  return <animated.div style={props}>{children}</animated.div>;
};

export default FadeInFromBottom;
```

上記で作成したアニメーションコンポーネントの使用例は、以下のとおりです。

```ts:App.tsx
import reactLogo from "./assets/react.svg";
import FadeInFromBottom from "./FadeInFromBottom";

function App() {

  return (
    <>
      <FadeInFromBottom>
        <img src={reactLogo} className="logo react" alt="React logo" />
      </FadeInFromBottom>
    </>
  );
}

export default App;

```

実際に動かしてみると、以下のようなイメージになります。

![image.png](/images/react-animation-infro/fadeInFromBottom.gif =600x)

実際のプロダクトに組み込んだときのイメージも紹介します。
チャット機能のファイルアップロードで、ファイルをドラッグ&ドロップする際にアイコンが上からふわっと浮き上がって表示されて、「これからファイルをアップロードするよ！」感が演出できました。
ちょっとした遊び心・ひと手間で、アニメーションをつけるだけでなんかイケてる感じになりませんか？

![image.png](/images/react-animation-infro/fadeInFromBottom2.gif =600x)

### フラグの ON・OFF でアニメーションさせる

任意のタイミングでアニメーションを実行させたいケースがあると思います。以下では、フラグの ON・OFF によって、状態を変化させるアニメーションを作りたいと思います。

前回と同様に、`useSpring` Hooks を利用するところは同じですが、今度は、CSS の各種設定として `height`、`opacity` の値が `isOpen` フラグの ON・OFF で変化するように記述しています。このように記述することで、`isOpen` が true になったときに、指定した高さに開き、opacity も 0.4 から 1 に変化するように設定しています。

この設定でどのようなアニメーションになるか想像してみてください（答えは下）

```ts:AccordionOpenAnimation.tsx
import { useSpring, animated } from "@react-spring/web";

type Props = {
  height: number;
  isOpen: boolean;
  onHeightChange?: (height: number) => void;
  onClose?: () => void;
  children: React.ReactNode;
};

/**
 * アコーディオン用のアニメーション設定
 * isOpen:trueの時に開く
 */
export const AccordionOpenAnimation = ({
  height,
  isOpen,
  onHeightChange,
  onClose,
  children,
}: Props) => {
  const accordionStyles = useSpring({
    // isOpen が true のときに高さを自動計算・opacity を 1 に
    // false のときは height: 0, opacity: 0 で閉じる
    height: isOpen ? height : 0,
    opacity: isOpen ? 1 : 0.4,
    // 下に向かって広がるようにするので overflow は必ず隠す
    overflow: "hidden",
    config: {
      tension: 500,
      friction: 40,
    },
    // アニメーションが終了したタイミングで呼ばれるコールバック
    onRest: () => {
      if (!isOpen) {
        onClose && onClose();
      }
    },
    onChange: (currentValues) => {
      // 現在の高さを親コンポーネントに渡す
      onHeightChange && onHeightChange(Number(currentValues.value.height));
    },
  });

  return <animated.div style={accordionStyles}>{children}</animated.div>;
};
```

答えは、アコーディオンのアニメーションです。

![image.png](/images/react-animation-infro/accordionOpenAnimation.gif =600x)

### おまけ: `useSpring` が用意しているコールバック関数

「フラグの ON・OFF でアニメーションさせる」先ほどのコードをよく見てみると、以下の 2 種類のコールバック関数が存在することがわかります。

- `onRest`: アニメーションが開始・終了したときに発火するコールバック関数
- `onChange`: アニメーション中で高さなどの値の変更を返してくれるコールバック関数

アニメーション終了時やアニメーション中に何らかの処理を挟みたいときに便利です。

```ts:AccordionOpenAnimation.tsx
onRest: () => {
  if (!isOpen) {
     onClose && onClose();
  }
},
onChange: (currentValues) => {
  // 現在の高さを親コンポーネントに渡す
  onHeightChange && onHeightChange(Number(currentValues.value.height));
},
```

## 終わりに

どうだったでしょうか？思ったより簡単にアニメーションを実装できると思いませんでしたか？
アニメーションをつけるかどうかは非機能要件で、あってもなくても動作に影響しないのですが、アニメーションをつけるだけで見栄えがグググッと上がると思っています。

あなたのプロダクトにも隠し味として「アニメーション」をつけてみませんか？
