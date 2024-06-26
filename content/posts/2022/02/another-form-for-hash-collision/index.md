---
title: "続・ランダム生成IDの衝突確率について"
date: 2022-02-10T17:25:18+09:00
math: true
tags:
  - 数学
---

[前回](/posts/2022/02/hash-collision) では数学的な側面として誕生日パラドックスに関する確率を求めた。

{{<katex display>}}{{</katex>}}

$$ P_{\text{nc}}(n) = \frac{M}{M}\frac{M-1}{M} \dots \frac{M-(n-1)}{M} = \frac{M!}{M^n (M-n)!} $$

$$ P_{\text{c}}(n) = 1 - P_{\text{nc}}(n) = 1 - \frac{M!}{M^n (M-n)!} $$

これは余事象である「一度も衝突しない確率」を求めることで計算したが、余事象を使わずに求めることができるだろうか？

## 余事象を使わない方法

「 \\(n\\) 回目までに衝突が発生する確率」を求めるには

- 1回目に「初めて」衝突する確率 \\(P_\text{fc}(1)\\)( = 0 )
- 2回目に「初めて」衝突する確率 \\(P_\text{fc}(2)\\)
- ...
- \\(n\\) 回目に「初めて」衝突する確率 \\(P_\text{fc}(n)\\)
  
をそれぞれ求めて足し合わせれば良い。
注意すべきポイントはそれぞれの「初めて衝突する事象」が重複していないことである。

具体的に求めていこう。

\\(n = 1\\) ならばどれが選ばれても衝突しないので、

$$ P_\text{fc}(1) = \frac{0}{M} = 0$$

となる。次に \\(n = 2\\) ならば、1つ目は何を選んでもよく、最初に選んだ1つと一致した場合なので、

$$ P_\text{fc}(2) = \frac{M}{M}\frac{1}{M} = \frac{1}{M}$$

となる。続いて、 \\(n = 3\\) ならば、2つ目までは衝突せず、先に選ばれた2つと一致した場合なので、

$$ P_\text{fc}(3) = \frac{M}{M}\frac{M-1}{M}\frac{2}{M}$$

となる。これを \\(n\\) の場合について考えると

$$ P_\text{fc}(n) = \frac{M!}{M^{n-1}(M-(n-1))!}\frac{n-1}{M}$$

となる。したがって求めるべき \\(n\\) 回目までに衝突が発生する確率は

$$ P_{\text{c}}'(n) = \sum_{k=1}^n P_\text{fc}(k) = \sum_{k=1}^n \frac{M!}{M^{k-1}(M-(k-1))!}\frac{k-1}{M}$$

となるはずである。

はたして、 \\( P_{\text{c}}(n) = P_{\text{c}}'(n)\\) となるだろうか？（一致しないとおかしいはずだが）

## 一致することを示す

一致することを示そう。

まず、 \\( P_\text{nc}(k)\\) のときの数式とよく見比べて、形の違う部分 \\(\frac{k-1}{M}\\) に注目し次のように変形する。

$$ \frac{k-1}{M} = \frac{M-M+k-1}{M} = 1 - \frac{M-(k-1)}{M}$$

これにより、 \\(P_\text{fc}(k)\\) は次のように変形できる。

$$
\begin{align*}
P_\text{fc}(k) &=  \frac{M!}{M^{k-1}(M-(k-1))!}\bigg\\{1 - \frac{M-(k-1)}{M}\bigg\\} \\\\
&= \frac{M!}{M^{k-1}(M-(k-1))!} - \frac{M!}{M^k(M-k)!}
\end{align*}
$$

したがって、総和である \\(P_{\text{c}}'(n)\\) は

$$ P_{\text{c}}'(n) = \sum_{k=1}^n \bigg\\{ \frac{M!}{M^{k-1}(M-(k-1))!} - \frac{M!}{M^k(M-k)!} \bigg\\} $$

となるが、総和の各項は次のように打ち消すことができる。

$$
\begin{align*}
P_\text{fc}(k) + P_\text{fc}(k+1) &= \frac{M!}{M^{k-1}(M-(k-1))!} - \frac{M!}{M^k(M-k)!} + \frac{M!}{M^{k}(M-k)!} - \frac{M!}{M^{k+1}(M-(k+1))!} \\\\ &= \frac{M!}{M^{k-1}(M-(k-1))!} - \frac{M!}{M^{k+1}(M-(k+1))!}
\end{align*}
$$

結果として最初と最後だけが残り、次の結果が得られる。

$$
\begin{align*}
P_{\text{c}}'(n) &= \sum_{k=1}^n \bigg\\{ \frac{M!}{M^{k-1}(M-(k-1))!} - \frac{M!}{M^k(M-k)!} \bigg\\} \\\\
&= 1 - \frac{M!}{M^n(M-n)!} \\\\
&= P_{\text{c}}(n)
\end{align*}
$$

よって、一致することが示せた。

## 別の考え方

\\(n\\) 回目で初めて衝突する確率 \\(P_\text{fc}(n)\\) から \\(n\\) 回目までに衝突が発生する確率 \\(P_\text{c}(n)\\) を計算してみたが、逆に考えることもできる。

\\(P_\text{fc}(k)\\) の変形後の数式を更に変形すると( \\(P_\text{c}(0)\\) は定義されていないため) \\(k \geq 2\\) の場合について次のことがわかる。

$$
\begin{align*}
P_\text{fc}(k) &= \frac{M!}{M^{k-1}(M-(k-1))!} - \frac{M!}{M^k(M-k)!} \\\\
&= \bigg(1 - \frac{M!}{M^k(M-k)!}\bigg) - \bigg(1 - \frac{M!}{M^{k-1}(M-(k-1))!}\bigg) \\\\
&= P_\text{c}(k) - P_\text{c}(k-1)
\end{align*}
$$

よく考えればわかることだが、 \\(n\\) 回目で「初めて」衝突する場合というのは「\\(n\\) 回目までに衝突が起こる場合」から「 \\(n-1\\) 回目までに衝突が発生する場合」を取り除いたものに等しく、この変形は納得の行く結果となる。

この結果から改めて \\(P_{\text{c}}'(n)\\) を計算してみると同様の結果が得られる。

$$
\begin{align*}
P_{\text{c}}'(n) &= \sum_{k=2}^n\\{P_\text{c}(k) - P_\text{c}(k-1)\\} + P_\text{fc}(1) \\\\
&= P_\text{c}(k) - P_\text{c}(1) + P_\text{fc}(1) \\\\
&= P_\text{c}(k)
\end{align*}
$$
