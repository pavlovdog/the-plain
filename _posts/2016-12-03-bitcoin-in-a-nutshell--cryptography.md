<h3>Introduction</h3>
<p>По сути, вся криптография, используемая в сети Bitcoin - это так называемая «Elliptic curve cryptography (ECC)». Этот тип криптографии основывается на использовании некоторой особой, довольно интересной функции - эллиптической кривой (не путать с эллипсом!).</p>

<p>В данной статье я расскажу, что это за функция, чем она примечательна, что такое приватные / публичные ключи, как это все выглядит с точки зрения Bitcoin и постараюсь привести столько примеров, сколько смогу написать.</p>

<p><img src="https://habrastorage.org/files/a27/ac5/0ad/a27ac50ad5454e3283b5b25795233f59.jpg" alt="hackerman"></p>
<cut />
<h3>Table of content</h3>
<ol>
<li>Elliptic curve</li>
<li>Elliptic curve over a finite field</li>
<li>SECP256k1</li>
<li>ECDSA</li>
<li>Private key</li>
<li>Public key</li>
<li>Private key formats</li>
<li>Public key formats</li>
<li>Links</li>
</ol>
<h3>Eliptic curve</h3>
<blockquote>
<p>Эллипти́ческая крива́я над полем <img src="//tex.s2cms.ru/svg/K" alt="K" /> — неособая кубическая кривая на проективной плоскости над <img src="//tex.s2cms.ru/svg/%20%7B%5Chat%20%7BK%7D%7D%20" alt=" {\hat {K}} " /> (алгебраическим замыканием поля <img src="//tex.s2cms.ru/svg/%20K%20" alt=" K " />), задаваемая уравнением 3-й степени с коэффициентами из поля <img src="//tex.s2cms.ru/svg/%20K%20" alt=" K " /> и «точкой на бесконечности» - <a href="https://ru.wikipedia.org/wiki/%D0%AD%D0%BB%D0%BB%D0%B8%D0%BF%D1%82%D0%B8%D1%87%D0%B5%D1%81%D0%BA%D0%B0%D1%8F_%D0%BA%D1%80%D0%B8%D0%B2%D0%B0%D1%8F">Wikipedia</a></p>
</blockquote>
<p>Если на пальцах, то эллиптическая кривая - это некоторая довольно простая функция, как правило записываемая в виде так называемой формы Вейерштрасса: <img src="//tex.s2cms.ru/svg/%20y%5E%7B2%7D%3Dx%5E%7B3%7D%2Bax%2Bb%20" alt=" y^{2}=x^{3}+ax+b " /></p>

<p>В зависимости от значений параметров <img src="//tex.s2cms.ru/svg/a" alt="a" /> и <img src="//tex.s2cms.ru/svg/b" alt="b" />, график данной функции может выглядеть по разному:</p>

<p><img src="https://habrastorage.org/files/ee8/746/c92/ee8746c9232b4033afd629279d5a3a8b.png" alt="elliptic curves"></p>

<p>Скрипт для отрисовки графика на Python:</p>
<pre><code>import numpy as np
import matplotlib.pyplot as plt

def main():
    a = -1
    b = 1

    y, x = np.ogrid[-5:5:100j, -5:5:100j]
    plt.contour(x.ravel(), y.ravel(), pow(y, 2) - pow(x, 3) - x * a - b, [0])
    plt.grid()
    plt.show()

if __name__ == '__main__':
    main()
</code></pre>
<p>Если верить wiki, то впервые эта функция засветилась еще в древней Греции, где Диофант пытался найти решения уравнения <img src="//tex.s2cms.ru/svg/y(6-y)%3Dx%5E%7B3%7D-x" alt="y(6-y)=x^{3}-x" />. <a href="https://www.wolframalpha.com/input/?i=6y%20-%20y%5E2%20=%20%283y-1%29%5E3%20-%203y%20%2b%201">Таки решил</a> с помощью подстановки <img src="//tex.s2cms.ru/svg/x%3D3y-1" alt="x=3y-1" />. Чуть позже, в 17 веке Ньютон также заинтересовался кубическими кривыми, и его открытия во многом привели в формулам сложения точек на эллиптической кривой, с которыми мы сейчас познакомимся. Здесь и в дальнейшем мы будем рассматривать некоторую эллиптическую кривую <img src="//tex.s2cms.ru/svg/%5Calpha" alt="\alpha" />.</p>

<p>Пусть есть две точки <img src="//tex.s2cms.ru/svg/P%2C%20Q%20%5Cin%20%5Calpha" alt="P, Q \in \alpha" />. Их суммой называется точка <img src="//tex.s2cms.ru/svg/R%20%5Cin%20%5Calpha" alt="R \in \alpha" />, которая в простейшем случае определяется следующим образом: проведем прямую через <img src="//tex.s2cms.ru/svg/P" alt="P" /> и <img src="//tex.s2cms.ru/svg/Q" alt="Q" /> - она пересечет кривую <img src="//tex.s2cms.ru/svg/%5Calpha" alt="\alpha" /> в единственной точке, назовем ее <img src="//tex.s2cms.ru/svg/-R" alt="-R" />. Поменяв <img src="//tex.s2cms.ru/svg/y" alt="y" /> координату точки <img src="//tex.s2cms.ru/svg/-R" alt="-R" /> на противоположенную по знаку, мы получим точку <img src="//tex.s2cms.ru/svg/R" alt="R" />, которую и будем называть суммой <img src="//tex.s2cms.ru/svg/P" alt="P" /> и <img src="//tex.s2cms.ru/svg/Q" alt="Q" />, то есть <img src="//tex.s2cms.ru/svg/P%20%2B%20Q%20%3D%20R" alt="P + Q = R" />.</p>

<p><img src="http://bilmuh.yasar.edu.tr/wp-content/uploads/2015/06/gamze-orhon1.jpg" alt="ellitic_curve_addiction"></p>

<p>Считаю необходимым отметить, что мы именно <strong>вводим</strong> такую операцию сложения - если вы будете складывать точки в привычном понимании, то есть складывая соответствующие координаты, то получите совсем другую точку <img src="//tex.s2cms.ru/svg/R'%20(x_1%20%2B%20x_2%2C%20y_1%20%2B%20y_2)" alt="R' (x_1 + x_2, y_1 + y_2)" />, которая не имеет ничего общего с <img src="//tex.s2cms.ru/svg/R" alt="R" /> или <img src="//tex.s2cms.ru/svg/-R" alt="-R" />, и скорее всего вообще не лежит на кривой <img src="//tex.s2cms.ru/svg/%5Calpha" alt="\alpha" />.</p>

<p>Самые сообразительные уже задались вопросом - а что будет, если например провести прямую через две точки, имеющие координаты вида <img src="//tex.s2cms.ru/svg/P%20(a%2C%20b)%24%20%D0%B8%20%24Q%20(a%2C%20-b)" alt="P (a, b)$ и $Q (a, -b)" />, то есть прямая, проходящая через них, будет параллельна оси ординат (третий кадр на картинке ниже).</p>

<p><img src="https://habrastorage.org/files/688/4f9/518/6884f95189be44acae28a94d578c4191.png" alt="elliptic_curve_parallel"></p>

<p>Несложно видеть, что в этом случае отсутствует третье пересечение в кривой <img src="//tex.s2cms.ru/svg/%5Calpha" alt="\alpha" />, которое мы называли <img src="//tex.s2cms.ru/svg/-R" alt="-R" />. Для того, чтобы избежать этого казуса - введем так называемую «точку в бесконечности» (point of infinity), обозначаемую обычно <img src="//tex.s2cms.ru/svg/O" alt="O" /> или просто <img src="//tex.s2cms.ru/svg/0" alt="0" />, как на картинке. И будем говорить, что в этом случае <img src="//tex.s2cms.ru/svg/P%20%2B%20Q%20%3D%20O" alt="P + Q = O" />.</p>

<p>Особый интерес для нас будет представлять случай, когда мы хотим сложить точку саму с собой (2 кадр, точка <img src="//tex.s2cms.ru/svg/Q" alt="Q" />). В этом случае просто проведем касательную к точке <img src="//tex.s2cms.ru/svg/Q" alt="Q" /> и отразим полученную точку пересечения относительно <img src="//tex.s2cms.ru/svg/y" alt="y" />.</p>

<p>Теперь, легким движением руки, можно ввести операцию умножения точки на какое-то <img src="//tex.s2cms.ru/svg/%5Cmathbb%7BN%7D" alt="\mathbb{N}" /> число. В результате получим новую точку <img src="//tex.s2cms.ru/svg/K%20%3D%20G*k" alt="K = G*k" />, то есть <img src="//tex.s2cms.ru/svg/K%20%3D%20G%20%2B%20G%20%2B%20...%20%2B%20G%2C%20%5C%20k" alt="K = G + G + ... + G, \ k" /> раз. С картинкой все должно стать вообще понятно:</p>

<p><img src="http://orm-chimera-prod.s3.amazonaws.com/1234000001802/images/msbt_0404.png" alt="elliptic_curve_multiplication"></p>

<h3>Elliptic curve over a finite field</h3>
<p>Непосредственно в нашем случае будет использоваться точно такая же кривая, только рассматриваемая над некоторым конечным полем <img src="//tex.s2cms.ru/svg/%7BF%7D%20_p%3D%5Cmathbb%7BZ%7D%20%2F%20%5Cmathbb%7BZ%7D_p%20%3D%20%5C%7B0%2C%201%2C%20...%2C%20p%20-%201%5C%7D" alt="{F} _p=\mathbb{Z} / \mathbb{Z}_p = \{0, 1, ..., p - 1\}" />, где <img src="//tex.s2cms.ru/svg/p" alt="p" /> - простое число. То есть</p>

<p align="center"><img align="center" src="//tex.s2cms.ru/svg/%0Ay%5E2%5C%20mod%5C%20p%20%3D%20x%5E2%20%2B%20ax%20%2B%20b%20%5C%20(mod%5C%20p)%0A" alt="
y^2\ mod\ p = x^2 + ax + b \ (mod\ p)
" /></p>
<p>Все названные свойства (сложение, умножение, точка в бесконечности) для такой функции остаются в силе, хотя если попробовать нарисовать данную функцию, то напоминать привычную эллиптическую кривую она будет лишь отдаленно (в лучшем случае). А понятие «касательной к функции в точке» вообще теряет всякий смысл, но это ничего страшного. Вот пример функции <img src="//tex.s2cms.ru/svg/y%5E2%20%3D%20x%5E3%20%2B%207" alt="y^2 = x^3 + 7" /> для <img src="//tex.s2cms.ru/svg/p%3D17" alt="p=17" />:</p>

<p><img src="https://habrastorage.org/files/b88/d43/1ec/b88d431ec74b422c995fe7a95fe4f4d0.png" alt="elliptic_curve_over_17"></p>

<p>А вот для <img src="//tex.s2cms.ru/svg/p%3D59" alt="p=59" />, тут вообще почти хаотичный набор точек. Единственное, что все еще напоминает о происхождении этого графика - так это симметрия относительно оси <img src="//tex.s2cms.ru/svg/X" alt="X" />.</p>

<p><img src="https://habrastorage.org/files/48a/74b/f17/48a74bf177fc4bb0bfa86678bccf6125.png" alt="elliptic_curve_59"></p>

<p><strong>P. S.</strong> Если вам хочется узнать, как в случае с кривой над конечным полем вычислить координаты точки <img src="//tex.s2cms.ru/svg/R%20(x_3%2C%20y_3)" alt="R (x_3, y_3)" />, зная координаты <img src="//tex.s2cms.ru/svg/P(x_1%2C%20y_1)%24%20%D0%B8%20%24Q(x_2%2C%20y_2)" alt="P(x_1, y_1)$ и $Q(x_2, y_2)" /> - рекомендую полистать <a href="http://www.slideshare.net/NikeshMistry1/introduction-to-bitcoin-and-ecdsa">«An Introduction to Bitcoin, Elliptic Curves and the Mathematics of ECDSA» by N. Mistry</a> или <a href="http://www.nicolascourtois.com/bitcoin/thesis_Di_Wang.pdf">«Secure Implementation of ECDSA Signatures in Bitcoin» by Di Wang</a>, там все подробно расписано, достаточно знать математику на уровне 8 класса.</p>

<p>Ну или можете поверить на слово - ниже приведена итоговая формула, где <img src="//tex.s2cms.ru/svg/%5Clambda%20%3D%20%5Cfrac%7By_2%20-%20y_1%7D%7Bx_2%20-%20x_1%7D%5C%20mod%5C%20p" alt="\lambda = \frac{y_2 - y_1}{x_2 - x_1}\ mod\ p" />:</p>

<p align="center"><img align="center" src="//tex.s2cms.ru/svg/%0A%5Cbegin%7Bcases%7D%20x_3%20%3D%20(%5Clambda%5E2%20-%20x_1%20-%20x_2)%5C%20mod%5C%20p%20%5C%5C%20y_3%20%3D%20(%5Clambda(x_1%20-%20x_3)%20-%20y_1)%5C%20mod%5C%20p%20%5Cend%7Bcases%7D%0A" alt="
\begin{cases} x_3 = (\lambda^2 - x_1 - x_2)\ mod\ p \\ y_3 = (\lambda(x_1 - x_3) - y_1)\ mod\ p \end{cases}
" /></p>
<p><strong>P. P. S.</strong> На случай, если мои примеры не удовлетворили ваш пытливый ум - <a href="https://cdn.rawgit.com/andreacorbellini/ecc/920b29a/interactive/modk-add.html">вот интерактивная рисовалка</a> самых разных кривых, поэкспериментируйте.</p>

<h3>SECP256k1</h3>
<p>Возвращаясь конкретно к Bitcoin - в нем используется кривая <a href="https://en.bitcoin.it/wiki/Secp256k1">SECP256k1</a>. Она имеет вид <img src="//tex.s2cms.ru/svg/y%5E2%20%3D%20x%5E3%20%2B%207" alt="y^2 = x^3 + 7" /> и рассматривается над полем <img src="//tex.s2cms.ru/svg/F_p" alt="F_p" />, где <img src="//tex.s2cms.ru/svg/p" alt="p" /> - очень большое простое число, а именно <img src="//tex.s2cms.ru/svg/2%5E%7B256%7D%20-%202%5E%7B32%7D%20-%202%5E%7B9%7D%20-%202%5E%7B8%7D%20-%202%5E%7B7%7D%20-%202%5E%7B6%7D%20-%202%5E%7B4%7D%20-%201" alt="2^{256} - 2^{32} - 2^{9} - 2^{8} - 2^{7} - 2^{6} - 2^{4} - 1" />.
Так же для SECP256k1 определена так называемая <strong>«base point»</strong> или <strong>«generator point»</strong> - это просто точка, как правило обозначаемая <img src="//tex.s2cms.ru/svg/G" alt="G" />, лежащая на данной кривой. Она нужна для создания публичного ключа, о котором будет рассказано ниже.</p>

<p>Еще определен параметр <img src="//tex.s2cms.ru/svg/n" alt="n" />, который называется порядком (order) генератора, то есть натуральное число, такое что <img src="//tex.s2cms.ru/svg/n*G%3DO" alt="n*G=O" />.</p>

<p>Простой пример: используя Python, проверим, принадлжит ли точка <img src="//tex.s2cms.ru/svg/G%20(x%2C%20y)" alt="G (x, y)" /> кривой SECP256k1</p>
<pre><code>&gt;&gt;&gt; p = 115792089237316195423570985008687907853269984665640564039457584007908834671663
&gt;&gt;&gt; x = 55066263022277343669578718895168534326250603453777594175500187360389116729240
&gt;&gt;&gt; y = 32670510020758816978083085130507043184471273380659243275938904335757337482424
&gt;&gt;&gt; (x ** 3 + 7) % p == y**2 % p
True
</code></pre>
<h3>Digital signature</h3>
<blockquote>
<p>Электро́нная по́дпись (ЭП), Электро́нная цифровая по́дпись (ЭЦП) — реквизит электронного документа, полученный в результате криптографического преобразования информации с использованием закрытого ключа подписи и позволяющий проверить отсутствие искажения информации в электронном документе с момента формирования подписи (целостность), принадлежность подписи владельцу сертификата ключа подписи (авторство), а в случае успешной проверки подтвердить факт подписания электронного документа (неотказуемость) - <a href="https://ru.wikipedia.org/wiki/%D0%AD%D0%BB%D0%B5%D0%BA%D1%82%D1%80%D0%BE%D0%BD%D0%BD%D0%B0%D1%8F_%D0%BF%D0%BE%D0%B4%D0%BF%D0%B8%D1%81%D1%8C">Wikipedia</a></p>
</blockquote>
<p>Общая идея такова - Алиса хочет перевести 1 BTC Бобу. Для этого она создает сообщение типа:</p>
<pre><code>{
	&quot;from&quot; : 1FXySbm7jpJfHEJRjSNPPUqnpRTcSuS8aN, // Alice's address
	&quot;to&quot; : 1Eqm3z1yu6D4Y1c1LXKqReqo1gvZNrmfvN, // Bob's address
	&quot;amount&quot; : 1 // Send 1 BTC
}
</code></pre>
<p>Потом Алиса берет свой приватный ключ (что это такое я расскажу ниже), хэш сообщения и функцию вида <img src="//tex.s2cms.ru/svg/sign%5C_text(private%5C_key%2C%20text)" alt="sign\_text(private\_key, text)" />. На выходе она получает некоторую <strong>подпись</strong> (signature) своего сообщения - в случае ECDSA это будет пара целых чисел, для других алгоритмов подпись может выглядеть по другому. После этого она рассылает всем участникам сети исходное сообщение, подпись и свой публичный ключ. В результате, каждый Вася при желании сможет взять эту троицу, функцию вида <img src="//tex.s2cms.ru/svg/validate%5C_signature(public%5C_key%2C%20signature%2C%20text)" alt="validate\_signature(public\_key, signature, text)" /> и проверить, действительно ли владелец приватного ключа подписывал это сообщение или нет.</p>

<p><img src="https://habrastorage.org/files/b23/856/57d/b2385657dd3545998a0b0bc4af4a0fd5.png" alt="digital_signature_scheme"></p>

<h4>AHTUNG! В реальности процесс довольно существенно отличается от вышеописанного. Здесь я просто на пальцах показал, что из себя представляет электронно цифровая подпись и зачем она нужна. Реальный алгоритм описан в статье <a href="#">«Bitcoin in a nutshell. Transactions.»</a></h4>
<h3>Private key</h3>
<p><strong>Приватный ключ</strong> - это довольно общий термин и в различных алгоритмах электронной подписи могут использоваться различные типы приватных ключей.</p>

<p>Как вы уже могли заметить, в Bitcoin используется алгоритм ECDSA - в его случае приватный ключ это некоторое натуральное <img src="//tex.s2cms.ru/svg/256" alt="256" /> битное число, то есть самое обычное целое число от <img src="//tex.s2cms.ru/svg/1" alt="1" /> до <img src="//tex.s2cms.ru/svg/2%5E%7B256%7D" alt="2^{256}" />. Технически, даже число <img src="//tex.s2cms.ru/svg/123456" alt="123456" /> будет являться корректным приватным ключом, но очень скоро вы узнаете, что ваши монеты «принадлежат» вам ровно до того момента, как у злоумышленника окажется ваш приватный ключ, а значения типа <img src="//tex.s2cms.ru/svg/123456" alt="123456" /> очень легко перебираются.</p>

<p>Важно отметить, что на сегодняшний день перебрать все ключи невозможно в силу того, что <img src="//tex.s2cms.ru/svg/2%5E%7B256%7D" alt="2^{256}" /> - это фантастически большое число.</p>

<p>Постараемся его представить - согласно <a href="http://www.quickanddirtytips.com/education/math/how-many-grains-of-sand-are-on-earth%E2%80%99s-beaches">этой статье</a>, на всей Земле немногим меньше <img src="//tex.s2cms.ru/svg/10%5E%7B22%7D" alt="10^{22}" /> песчинок. Воспользуемся тем, что <img src="//tex.s2cms.ru/svg/2%5E%7B10%7D%20%E2%89%88%2010%5E3" alt="2^{10} ≈ 10^3" />, то есть <img src="//tex.s2cms.ru/svg/10%5E%7B22%7D%20%E2%89%88%202%5E%7B80%7D" alt="10^{22} ≈ 2^{80}" /> песчинок (будем оценивать сверху). А всего адресов у нас <img src="//tex.s2cms.ru/svg/2%5E%7B256%7D%24%2C%20%D1%82%D0%BE%20%D0%B5%D1%81%D1%82%D1%8C%20%24%E2%89%88%20%7B2%5E%7B80%7D%7D%5E3" alt="2^{256}$, то есть $≈ {2^{80}}^3" />.</p>

<p>Значит мы можем взять весь песок на Земле, превратить каждую песчинку в новую Землю, в получившей куче Земель каждую песчинку на каждой планете снова превратить в новую Землю - и суммарное число песчинок все равно будет в разы меньше числа возможных приватных ключей.</p>

<p>По этой же причине, большинство Bitcoin клиентов при создании приватного ключа просто берут <img src="//tex.s2cms.ru/svg/256" alt="256" /> случайных бит - вероятности коллизии пренебрежимо мала.</p>

<h4>Создание приватного ключа на Python, <a href="https://github.com/warner/python-ecdsa">ECDSA</a></h4>
<pre><code>&gt;&gt;&gt; import binascii
&gt;&gt;&gt; import ecdsa # sudo pip install ecdsa
&gt;&gt;&gt; private_key = ecdsa.SigningKey.generate(curve=ecdsa.SECP256k1)
&gt;&gt;&gt; private_key.to_string()
'\xceG\xc0J\tu&quot;\xd3;K\x00;%\xdd~\x8dyE\xeaR\xfa\x891\xfd\x9a\xa5[1Z9\xdcb'
&gt;&gt;&gt; binascii.hexlify(private_key.to_string()).decode('ascii')
u'5be674003f23d8bdc72a24281341a4f84ee9d5edd7890de2072c310160024fce'
</code></pre>
<h4>Создание приватного ключа с помощью bitcoin-cli</h4>
<pre><code>$ bitcoin-cli getnewaddress
14RVpC4su4PzSafjCKVWP2YBHv3f6zNf6U
$ bitcoin-cli dumpprivkey 14RVpC4su4PzSafjCKVWP2YBHv3f6zNf6U
L3SPdkFWMnFyDGyV3vkCjroGi4zfD59Wsc5CHdB1LirjN6s2vii9
</code></pre>
<h4>Создание приватного ключа для самых недоверчивых</h4>
<p>Если вы страдаете от периодических приступов паранойи, то можете создать себе криптостойкий приватный ключ сами - возьмите бумагу, ручку и подбросьте монетку 256 раз, записывая решку как 1, а орел как 0 .</p>

<h3>Public key</h3>
<p>Пусть <img src="//tex.s2cms.ru/svg/k" alt="k" /> - наш приватный ключ, <img src="//tex.s2cms.ru/svg/G" alt="G" /> - base point (некоторая всем известная точка на кривой SECP256k1), тогда публичный ключ <img src="//tex.s2cms.ru/svg/p%3DG*k" alt="p=G*k" />. То есть фактически, <strong>публичный ключ</strong> - это некоторая точка, лежащая на кривой SECP256k1. В простейшем случае он записывается с виде 64 байтов, где первые 32 байта - координата <img src="//tex.s2cms.ru/svg/x" alt="x" />, вторые - координата <img src="//tex.s2cms.ru/svg/y" alt="y" />.</p>

<p>Два важных нюанса. Во-первых, несложно видеть, что операция получения публичного ключа определена однозначно, то есть конкретному приватному ключу всегда соответствует один единственный публичный ключ. Во-вторых, обратная операция является вычислительно трудной и, в общем случае, получить приватный ключ из публичного можно только полным перебором первого. Это так называемая проблема дискретного логарифмирования и это именно то, что делает данный алгоритм криптостойким.</p>

<p>В следующей статье вы узнаете, что точно так же дело обстоит и со связкой публичный ключ / адрес.</p>

<p><img src="https://habrastorage.org/files/3ff/4e5/f93/3ff4e5f939a847b2aa40bfe4701f4bd9.png" alt="keys_to_address"></p>

<h4>Создание публичного ключа с помощью Python, ECDSA</h4>
<pre><code>&gt;&gt;&gt; import binascii
&gt;&gt;&gt; import ecdsa
&gt;&gt;&gt; private_key = ecdsa.SigningKey.generate(curve=ecdsa.SECP256k1)
&gt;&gt;&gt; public_key = private_key.get_verifying_key()
&gt;&gt;&gt; public_key.to_string()
'\xd5\xc0\x8f\x1b\xfc\x9c&amp;\xa5\xd1\x8f\xe9%Ny#\xde\xbb\xd3J\xfb\x92\xac#\xab\xfcc\x88\xd2e\x94F\xc1\xf0L\xcd\xeb\xb6w\xea\xab\xfe\xd9)Fc\xeey\xd7\x1bW\xcajkv\xbcG\xe6\xf8g\x0f\xe7Y\xd7F'
&gt;&gt;&gt; binascii.hexlify(public_key.to_string()).decode('ascii').upper()
u'D5C08F1BFC9C26A5D18FE9254E7923DEBBD34AFB92AC23ABFC6388D2659446C1F04CCDEBB677EAABFED9294663EE79D71B57CA6A6B76BC47E6F8670FE759D746'
&gt;&gt;&gt; # Для наглядности, покажу, как подписываются данные
&gt;&gt;&gt; private_key.sign(&quot;message&quot;)
&quot;\xe0\\\xdbc\xa7\x0e'\xe5\xd3\r\xbc\x9d(|\xcfR0\x15?\x0fC\x0c\x08P.\x9a\xf5J\xf4&lt;\\\xde\xd4\xca \xc4\xd4\x87p|\x03\x95f\xbeo\x02fn\xdc\xb7;yj+\xd0LL\xfd\xe6P$$\x85\xdf&quot;
&gt;&gt;&gt; signature = private_key.sign(&quot;message&quot;)
&gt;&gt;&gt; binascii.hexlify(signature).decode('ascii')
u'e35319a410c103389611b4d46c34103f3d5ee14906180d06461e7bcf2c296ef83de6dfadc193ea8093ccc40a95b2c77f4d2dd84ec068918c1878282e20044b4a'
&gt;&gt;&gt; public_key.verify(signature, &quot;message&quot;) # Иначе был бы exception
True
</code></pre>
<h4>Создание публичного ключа на C++, <a href="https://github.com/libbitcoin/libbitcoin">libbitcoin</a></h4>
<pre><code>#include &lt;bitcoin/bitcoin.hpp&gt;

int main() {
    // Private secret key.
    bc::ec_secret secret = bc::decode_hash(
    &quot;038109007313a5807b2eccc082c8c3fbb988a973cacf1a7df9ce725c31b14776&quot;);
    // Get public key.
    bc::ec_point public_key = bc::secret_to_public_key(secret);
    std::cout &lt;&lt; &quot;Public key: &quot; &lt;&lt; bc::encode_hex(public_key) &lt;&lt; std::endl;
}
</code></pre>
<p>Для компиляции и запуска используем (предварительно установив libbitcoin):</p>
<pre><code>$ g++ -o public_key &lt;filename&gt; $(pkg-config --cflags --libs libbitcoin)
$ ./public_key
Public key: 0202a406624211f2abbdc68da3df929f938c3399dd79fac1b51b0e4ad1d26a47aa
</code></pre>
<h3>Links</h3>
<ol>
<li><a href="https://ru.wikipedia.org/wiki/%D0%94%D0%B8%D1%81%D0%BA%D1%80%D0%B5%D1%82%D0%BD%D0%BE%D0%B5_%D0%BB%D0%BE%D0%B3%D0%B0%D1%80%D0%B8%D1%84%D0%BC%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5">Проблема дискретного логарифмирования</a></li>
</ol>