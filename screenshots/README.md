# スクリーンショットの案内 (Screenshots)

日本語 · [English below](#english)

このフォルダーには、性質の異なる二つの画面コレクションが入っています。混在して見えないように、
どちらが何なのかを先に明記します。

- **`ko/`: ARTEX 韓国語版の実際の UI キャプチャ。** このリポジトリが提供する成果物です。
  メイン文書の [`../README.md`](../README.md) と [`../README.en.md`](../README.en.md) が
  この画像を使っています。韓国語版の画面だけをご覧になる場合は、[`ko/`](ko/) フォルダーを開いてください。
- **ルートの `*.png`: アップストリーム原本(中国語版)の UI キャプチャ。** アップストリームのリポジトリと 1:1 で
  照合できるように**原本のまま保存**したもので、保存してある中国語 README
  [`../README.zh.md`](../README.zh.md) がこの画像を参照しています。韓国語・英語の文書では
  使用していません。
- **`wx.png`: アップストリーム原作者の WeChat 公式アカウントの告知画像。**
  [`../README.zh.md`](../README.zh.md) が原本のとおりにレンダリングされるよう、あわせて保存しました。
  韓国語版とは無関係です。

つまり、ルートに中国語の画面が先に見えるのはアップストリームを保存しているためであり、**韓国語版の画面は
[`ko/`](ko/) の中にあります。** このリポジトリがなぜ原本を一緒に置いているのか(ローカライズの透明性・アップストリームとの
照合)は、防御・検知というポジショニングと同じ文脈にあります。詳しい背景はルートの README を参照してください。

---

## English

This folder holds two separate sets of captures. To keep them from looking mixed, here
is what each one is.

- **`ko/` — real UI of the Korean edition of ARTEX.** This is what the fork delivers. The
  main docs [`../README.md`](../README.md) and [`../README.en.md`](../README.en.md) use
  these images. For the Korean edition's screens, open [`ko/`](ko/).
- **The `*.png` files in the root — upstream (Chinese edition) UI captures.** They are
  **preserved as-is** so the fork can be diffed against the upstream repository 1:1, and
  the preserved Chinese README [`../README.zh.md`](../README.zh.md) references them. The
  Korean and English docs do not use them.
- **`wx.png` — the upstream author's WeChat channel promo image.** Kept so that
  [`../README.zh.md`](../README.zh.md) still renders unchanged. It is unrelated to the
  Korean edition.

In short, the Chinese screens you see first in the root are there for upstream
preservation; the **Korean edition's screens live in [`ko/`](ko/).** See the root README
for why this fork keeps the originals alongside (localization transparency and upstream
comparison).
