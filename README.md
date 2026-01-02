\documentclass[12pt,a4paper]{article}

\usepackage[utf8]{inputenc}
\usepackage[T1]{fontenc}
\usepackage[turkish]{babel}
\usepackage{amsmath,amssymb}
\usepackage{geometry}
\geometry{margin=2.5cm}
\usepackage{hyperref}
\usepackage{url}
\usepackage{algorithm}
\usepackage{algpseudocode}
\usepackage{float}
\usepackage{booktabs} % Tabloların daha profesyonel görünmesi için
\usepackage{graphicx}
\usepackage{caption}

\title{EMÜ654 - Proje Final Raporu\\
\large Kardinalite Kısıtlı Portföy Optimizasyonu - Genetik Algoritma Yaklaşımı}
\author{Batıhan KONUK - N24141171}
\date{Ocak 2026}

\begin{document}
\maketitle

\newpage

\section*{Özet}

Bu proje, EMÜ654 \emph{Eniyileme için Sezgisel Yöntemler} dersi kapsamında, finans literatüründe NP-zor (NP-hard) bir problem olarak bilinen kardinalite kısıtlı portföy optimizasyonu (KKPO) problemini ele almaktadır. Çalışmada, Markowitz'in ortalama-varyans modeline varlık sayısı (kardinalite) ve yatırım ağırlığı sınırları eklenerek oluşturulan Karışık Tamsayılı Kuadratik Programlama (MIQP) modeli temel alınmıştır. Chang vd. (2000) \cite{chang2000} tarafından verilen CCMV/MIQP formülasyonu ve benzer çalışmalar (Deng-PSO \cite{deng2012}, Mansouri-ARO \cite{mansouri2021}) temel alınarak, Numba ile hızlandırılmış ve $\lambda$ değerleri boyunca paralel çalıştırılabilen bir Hibrit Genetik Algoritma (HGA) uygulanmıştır. Deneyler OR-Library veri setlerinden Port1–Port4 (Port1 - Hang Seng, Port2 - DAX 100, Port3 - FTSE 100, Port4 - S\&P 100) üzerinde ($K=10$, $e_i=0.01$, $d_i=1.0$) gerçekleştirilmiş; çözüm kalitesi Mean Percentage Error (MPE) ile, ayrıca GA amaç değeri MIQP amaç değeri ile kıyaslanarak değerlendirilmiştir. Sonuçlarda önerilen GA’nın MPE değerleri \%1.11–\%2.88 aralığında, MIQP’in MPE değerleri \%0.85–\%2.33 aralığında gözlenmiştir. Amaç fonksiyonu açısından GA–MIQP göreli farkı (Gap) \%0.31–\%2.50 aralığında kalmıştır. Süre karşılaştırmasında MIQP küçük örneklerde çok hızlıyken (ör. Port1), daha büyük/zor örneklerde süre artışı gösterebilmekte; önerilen GA ise veri setleri arasında daha istikrarlı süreler üretmektedir.

\newpage

\section{Giriş}

Modern portföy teorisi, Harry Markowitz'in 1952 tarihli çalışmasıyla temelleri atılan ve yatırımcıların belirli bir risk seviyesinde getirilerini maksimize etmeyi amaçladıkları bir çerçevedir \cite{markowitz1952}. Ancak klasik model, varlıkların istenildiği kadar küçük parçalara bölünebildiği ve portföyde tutulacak varlık sayısında bir sınır olmadığı varsayımına dayanır. Gerçek piyasa koşullarında ise işlem maliyetleri, yönetim zorlukları ve yasal düzenlemeler, yatırımcıları portföylerindeki varlık sayısını sınırlamaya (kardinalite kısıtı) ve her bir varlık için minimum/maksimum yatırım oranları belirlemeye iter.

Bu kısıtların eklenmesiyle problem, konveks yapısını kaybeder ve çözüm uzayı parçalı, süreksiz bir hal alır. Bu durum, problemi NP-zor sınıfına sokar ve klasik kuadratik programlama yöntemlerinin büyük ölçekli verilerde yetersiz kalmasına neden olur. Bu nedenle, literatürde Genetik Algoritmalar (GA), Tabu Arama (TS), Parçacık Sürü Optimizasyonu (PSO) ve Yapay Arı Kolonisi (ABC) gibi meta-sezgisel yöntemler ön plana çıkmıştır \cite{chang2000, deng2012}.

Bu çalışmada, literatürdeki standart veri setleri (OR-Library) kullanılarak, kardinalite kısıtlı portföy problemi için bir Genetik Algoritma tasarlanmış ve uygulanmıştır. Çalışmanın özgün katkısı, GA'nın değerlendirme/onarım bileşenlerinin \emph{Numba} ile hızlandırılması, $\lambda$ taramasının \emph{Joblib} ile paralel çalıştırılması ve \emph{steady-state} (kararlı durum) evrim şemasının uygulanmasıdır. Ayrıca literatürdeki farklı iyileştirme fikirleri (ör.\ Deng vd.\ \cite{deng2012}) incelenmiş ve karşılaştırma bölümünde referans alınmıştır.


\subsection{Deneysel Ortam ve Yazılım Özellikleri}

Bu çalışmada sunulan tüm algoritmalar ve modeller, \textbf{Python 3.12} programlama dili kullanılarak kodlanmış ve \textbf{Windows 11} işletim sistemi üzerinde çalışan, \textbf{Intel Core i7-12700H @ 2.10GHz} işlemci ve \textbf{32 GB} RAM belleğe sahip bir kişisel bilgisayarda test edilmiştir.

Hesaplamalı deneylerde kullanılan yazılım ve kütüphane detayları şöyledir:
\begin{itemize}
    \item \textbf{Kesin Çözüm (Exact):} Karışık Tamsayılı Kuadratik Programlama (MIQP) modellerinin çözümü için \textbf{Gurobi Optimizer (v12.0.2)} çözücüsü kullanılmıştır.
    \item \textbf{Sezgisel Yöntem (Heuristic):} Önerilen Hibrit Genetik Algoritma, Python ortamında geliştirilmiştir. Algoritmanın değerlendirme ve onarım fonksiyonları \textbf{Numba (JIT)} kütüphanesi ile makine koduna derlenerek hızlandırılmış, \textbf{Joblib} kütüphanesi kullanılarak çok çekirdekli (multi-core) paralel işleme tabi tutulmuştur.
    \item \textbf{Veri İşleme:} Vektör ve matris operasyonları için \textbf{NumPy} ve \textbf{Pandas} kütüphaneleri kullanılmıştır.
\end{itemize}

\section{Literatür Araştırması}

Kardinalite kısıtlı portföy optimizasyonu (KKPO) literatürü, problemin NP-zor doğası nedeniyle iki ana eksende gelişmiştir: (i) karışık tamsayılı kuadratik programlama (MIQP) gibi kesin (exact) çözüm yöntemleri ve (ii) meta-sezgisel (Genetik Algoritmalar, Parçacık Sürü Optimizasyonu vb.) tabanlı yaklaşık çözüm yöntemleri.

Chang vd.\ \cite{chang2000} çalışması, Markowitz modeline kardinalite ve alt/üst sınır kısıtlarını ekleyerek problemi standart bir test seti (OR-Library) üzerinde tanımlayan ve genetik algoritma, tabu arama, simüle tavlama yöntemlerini kıyaslayan öncü çalışmalardan biridir.

Son yıllarda meta-sezgisel tarafta, özellikle hibrit ve gelişmiş genetik algoritmalar öne çıkmaktadır. Kalaycı vd.\ \cite{kalayci2020}, kardinalite kısıtlı portföy optimizasyonu için yapay arı kolonisi ve yerel arama bileşenlerini birleştiren etkin bir hibrit meta-sezgisel önererek, büyük ölçekli problemlerde klasik yöntemlere kıyasla daha iyi çözümler üretebildiğini göstermiştir. Deng vd.\ \cite{deng2012}, gelişmiş bir Parçacık Sürü Optimizasyonu (PSO) algoritması sunmuş ve özellikle "en zayıf varlığı eleme" stratejisinin çözüm kalitesini artırdığını vurgulamıştır. Mansouri vd.\ \cite{mansouri2021} ise Eşeysiz Üreme Optimizasyonu (ARO) adını verdikleri yeni bir meta-sezgisel ile aynı veri setleri üzerinde rekabetçi sonuçlar elde etmişlerdir.

Literatürde ayrıca, algoritma performanslarını zaman kısıtlı bir çerçevede inceleyen çalışmalar da mevcuttur. Nikiporenko \cite{nikiporenko2023}, GA, tabu arama ve simüle tavlama algoritmalarını belirli bir hesaplama süresi altında sistematik olarak karşılaştırmış ve algoritmaların yakınsama hızlarını analiz etmiştir. Daha güncel çalışmalarda ise Guarino vd.\ \cite{guarino2024} ve Muteba Mwamba vd.\ \cite{mwamba2025}, NSGA-II ve NSGA-III gibi çok amaçlı evrimsel algoritmaları kullanarak risk ve getiri hedeflerini eş zamanlı optimize eden yaklaşımlar önermiştir.

Bu proje, söz konusu literatürle uyumlu olarak, Chang vd.\ \cite{chang2000} tarafından standartlaştırılan OR-Library veri setleri üzerinde, onarım (repair) tabanlı ve \emph{steady-state} evrim şemasına sahip bir genetik algoritma yaklaşımı sunmaktadır. Deng vd.\ \cite{deng2012} ve Mansouri vd.\ \cite{mansouri2021} gibi çalışmalar ise bu raporda karşılaştırma amacıyla ele alınmıştır. Çalışma, literatürdeki mevcut yöntemlerden farklı olarak, hem geometrik hata (Mean Percentage Error) hem de matematiksel amaç fonksiyonu değeri (Objective Value) bazında kesin çözüm (MIQP) ile kapsamlı bir kıyaslama sunmayı hedeflemektedir.


\section{Matematiksel Model}

Bu çalışmada, Chang vd. \cite{chang2000} tarafından önerilen Kardinalite Kısıtlı Ortalama-Varyans (CCMV) modeli temel alınmıştır. Model, klasik Markowitz çerçevesine yatırımcıların pratik kısıtlarını (sınırlı varlık sayısı ve alt/üst yatırım limitleri) entegre eden Karışık Tamsayılı Kuadratik Programlama (MIQP) yapısındadır.

\subsection{Notasyon}

Modelde kullanılan parametreler ve karar değişkenleri aşağıda tanımlanmıştır:

\textbf{Kümeler ve Parametreler:}
\begin{itemize}
    \item $n$: Yatırım evrenindeki toplam varlık sayısı.
    \item $\mu_i$: $i$. varlığın beklenen getiri oranı.
    \item $\sigma_{ij}$: $i$. ve $j$. varlıklar arasındaki kovaryans.
    \item $K$: Portföyde tutulması istenen varlık sayısı (Kardinalite).
    \item $e_i$: $i$. varlık için izin verilen minimum yatırım oranı (alt sınır).
    \item $d_i$: $i$. varlık için izin verilen maksimum yatırım oranı (üst sınır).
    \item $\lambda$: Riskten kaçınma parametresi ($0 \le \lambda \le 1$).
\end{itemize}

\textbf{Karar Değişkenleri:}
\begin{itemize}
    \item $x_i$: $i$. varlığa yatırılan sermaye oranı (Sürekli değişken, $0 \le x_i \le 1$).
    \item $z_i$: $i$. varlığın portföye seçilip seçilmediğini gösteren ikili değişken ($z_i \in \{0, 1\}$).
\end{itemize}

\subsection{Amaç Fonksiyonu}

Modelin amacı, yatırımcının risk algısına ($\lambda$) bağlı olarak portföyün toplam riskini minimize ederken beklenen getirisini maksimize etmektir. Bu iki zıt hedef, aşağıdaki ağırlıklı toplam fonksiyonu ile tek bir minimizasyon problemine indirgenmiştir:

\begin{equation} \label{eq:objective}
    \min_{\mathbf{x},\mathbf{z}} \quad f(\mathbf{x};\lambda) = 
    \lambda \underbrace{\left(\sum_{i=1}^n \sum_{j=1}^n x_i x_j \sigma_{ij}\right)}_{\text{Portföy Riski (Varyans)}} 
    - (1-\lambda) \underbrace{\left(\sum_{i=1}^n x_i \mu_i\right)}_{\text{Portföy Getirisi}}
\end{equation}

Burada $\lambda=1$ durumu sadece riski minimize ederken, $\lambda=0$ durumu sadece getiriyi maksimize etmeye odaklanır. $\lambda$ parametresinin $[0,1]$ aralığında değiştirilmesiyle Etkin Sınır (Efficient Frontier) üzerindeki farklı portföyler elde edilir.

\subsection{Kısıtlar}

Modelin tabi olduğu kısıtlar ve açıklamaları şöyledir:

\textbf{1. Bütçe Kısıtı:} Toplam yatırım oranlarının 1'e (%100) eşit olmasını sağlar.
\begin{equation} \label{eq:budget}
    \sum_{i=1}^n x_i = 1
\end{equation}

\textbf{2. Kardinalite Kısıtı:} Portföyde tam olarak $K$ adet varlığın bulunmasını zorunlu kılar.
\begin{equation} \label{eq:cardinality}
    \sum_{i=1}^n z_i = K
\end{equation}

\textbf{3. Yatırım Sınırları (Alt ve Üst Limitler):} Eğer bir varlık portföye seçildiyse ($z_i=1$), o varlığın ağırlığı $e_i$ ve $d_i$ sınırları arasında olmalıdır. Eğer seçilmediyse ($z_i=0$), ağırlığı 0 olmalıdır. Bu kısıt, mantıksal ilişkiyi kurar:
\begin{equation} \label{eq:bounds}
    e_i z_i \le x_i \le d_i z_i, \quad \forall i = 1,\dots,n
\end{equation}

\textbf{4. Değişken Tanımları:}
\begin{equation} \label{eq:domains}
    x_i \ge 0, \quad z_i \in \{0, 1\}, \quad \forall i = 1,\dots,n
\end{equation}
\section{Metodoloji: Hibrit Genetik Algoritma}

Geliştirilen algoritma, Chang vd. \cite{chang2000}'nin temel iskeletini korumakla birlikte, hesaplama verimliliği ve çözüm kalitesini artırmak için modern tekniklerle donatılmıştır.

\subsection{Çözüm Temsili ve Onarım (Repair) Mekanizması}
Her kromozom iki bileşenden oluşur: Seçilen $K$ adet varlığın indekslerini tutan bir küme ($Q$) ve bu varlıkların ağırlıklarını belirlemek için kullanılan reel sayı dizisi ($S$). Ancak rastgele üretilen veya çaprazlama sonucu oluşan ağırlıklar, Denklem \eqref{eq:budget} ve \eqref{eq:bounds}'u sağlamayabilir. Bu nedenle, Chang vd. \cite{chang2000} tarafından önerilen iteratif onarım (repair) fikri esas alınarak, her birey değerlendirme aşamasında $\textsc{EvaluateWithRepair}$ fonksiyonu ile kısıtları sağlayacak şekilde onarılmaktadır.


\textbf{Hızlandırma:} Bu onarım fonksiyonu, Python döngülerinin yavaşlığından kurtulmak için \emph{Numba JIT (Just-In-Time)} derleyicisi ile makine koduna derlenmiş ve vektörel işlemlerle optimize edilmiştir. Bu sayede değerlendirme fonksiyonu binlerce kez çağrıldığında dahi yüksek hız korunmuştur.

\subsection{Genetik Operatörler}
\begin{itemize}
    \item \textbf{Seçim:} İkili turnuva (Binary Tournament) yöntemi.
    \item \textbf{Çaprazlama:} "Uniform Crossover".
    \item \textbf{Mutasyon:} Belirli olasılıkla seçili varlıklardan birinin $S$ değeri $0.9$ veya $1.1$ ile çarpılarak hafifçe arttırılır veya azaltılır.
\end{itemize}


\subsection{Paralelizasyon}
Etkin sınırı (Efficient Frontier) oluşturmak için $\lambda$ parametresi $[0,1]$ aralığında 50 eşit parçaya bölünmüştür. Her bir $\lambda$ değeri birbirinden bağımsız bir optimizasyon problemi olduğu için, \emph{Joblib} kütüphanesi kullanılarak bu problemler işlemcinin tüm çekirdeklerinde paralel olarak çözdürülmüştür.

\subsection{Önerilen Hibrit Genetik Algoritma ve Sözde Kodu}

Bu çalışmada uygulanan yaklaşım, kardinalite kısıtlı portföy optimizasyonu probleminde genetik algoritmanın (GA) keşif yeteneğini, Chang vd.\ \cite{chang2000} tarafından vurgulanan onarım (repair) mantığıyla birleştiren hibrit bir çerçevedir. Hibritlik, her bireyin ham karar değişkenleri $(Mask, S)$ üzerinden üretilip, değerlendirme aşamasında $\textsc{EvaluateWithRepair}$ fonksiyonu ile (ağırlık sınırları ve toplam ağırlık koşulu gibi kısıtları sağlayacak biçimde) onarılarak amaç fonksiyonunun hesaplanmasıyla sağlanmaktadır. Evrimsel süreçte \emph{Steady-State} (Kararlı Durum) yaklaşımı benimsenmiş; her iterasyonda tek bir çocuk birey üretilmiş ve yalnızca popülasyondaki en kötü bireyden daha iyi olması durumunda popülasyona dahil edilmiştir.

Algoritmada kullanılan temel semboller ve parametreler Tablo \ref{tab:algo_params}'de özetlenmiştir.

\begin{table}[H]
\centering
\caption{GA Sözde Kodunda Kullanılan Semboller ve Açıklamaları}
\label{tab:algo_params}
\begin{tabular}{ll}
\toprule
\textbf{Sembol} & \textbf{Açıklama} \\
\midrule
$P$ & Popülasyon (bireyler kümesi) \\
$PopSize$ & Popülasyon büyüklüğü \\
$Mask$ & Seçilen varlıkları gösteren ikili vektör ($z_i$) \\
$S$ & Seçili varlıklar için ham (normalize edilmemiş) ağırlık parametreleri \\
$W$ & Onarım sonrası elde edilen portföy ağırlıkları dizisi ($\mathbf{w}$) \\
$Fits$ & Uygunluk/amaç fonksiyonu değerleri dizisi \\
$T_{iter}$ & Toplam iterasyon sayısı \\
$Child$ & Çaprazlama/mutasyon sonrası üretilen çocuk birey $(Mask,S)$ \\
$p_{val}$ & Değer mutasyonu olasılığı (bu çalışmada $0.5$) \\
$\lambda$ & Risk-getiri dengesini belirleyen ağırlık parametresi \\
$\textsc{EvaluateWithRepair}$ & $(Mask,S)$'yi onarıp $\mathbf{w}$ ve amaç değerini üreten fonksiyon \\
$\textsc{FixCardinality}$ & Kardinaliteyi ($K$) sağlayacak şekilde $Mask$ ve $S$'yi onaran fonksiyon \\
\bottomrule
\end{tabular}
\end{table}

Algoritmanın genel akışı aşağıdaki adımları izler:
\begin{enumerate}
    \item \textbf{Başlatma:} Rastgele $K$ varlık seçilerek başlangıç popülasyonu oluşturulur. Her birey $\textsc{EvaluateWithRepair}$ ile onarılır ve amaç değeri hesaplanır.
    \item \textbf{Seçim ve Çaprazlama:} İkili turnuva yöntemiyle seçilen iki ebeveynden uniform çaprazlama ile bir çocuk birey üretilir.
    \item \textbf{Mutasyon:} Belirli olasılıkla ($p_{val}$) çocuk bireyde seçili varlıklardan birinin $S$ değeri $0.9$ veya $1.1$ ile çarpılarak hafifçe arttırılır veya azaltılır.
    \item \textbf{Kardinalite Onarımı:} $\textsc{FixCardinality}$ ile çocuk bireyin tam olarak $K$ varlık seçmesi sağlanır.
    \item \textbf{Değerlendirme ve Yerine Koyma:} Çocuk birey onarılıp değerlendirilir. Popülasyondaki en kötü bireyden daha iyi ise popülasyona alınır (\emph{steady-state replacement}). Aynı zamanda global en iyi çözüm güncellenir.
\end{enumerate}

Algoritmanın ayrıntılı sözde kodu Algoritma \ref{alg:ga_matches_code}'da verilmiştir.

\begin{algorithm}[H]
\caption{Kardinalite Kısıtlı Portföy Optimizasyonu için Steady-State GA}
\label{alg:ga_matches_code}
\begin{algorithmic}[1]
\State \textbf{Girdi:} $\boldsymbol{\mu}$, $\Sigma$, $K$, $e_{min}, d_{max}$, $\lambda$
\State \textbf{Parametreler:} $PopSize$, $T_{iter}$, $p_{val}=0.5$
\State \textbf{Çıktı:} En iyi ağırlıklar $\mathbf{w}^*$ ve amaç değeri $f^*$
\Statex

\State \textbf{// Başlatma}
\For{$i=1$ \textbf{to} $PopSize$}
    \State $(P[i].Mask, P[i].S) \gets$ Rastgele $K$ varlık seç, $S$'yi rastgele üret
    \State $(Fits[i], W[i]) \gets \textsc{EvaluateWithRepair}(P[i].Mask, P[i].S, \lambda)$
\EndFor
\State $BestIdx \gets \arg\min(Fits)$
\State $\mathbf{w}^* \gets W[BestIdx]$, \quad $f^* \gets Fits[BestIdx]$
\Statex

\For{$t=1$ \textbf{to} $T_{iter}$}
    \State $p_1 \gets$ Binary tournament ile seç
    \State $p_2 \gets$ Binary tournament ile seç

    \State $(Child.Mask, Child.S) \gets$ Uniform crossover $(p_1, p_2)$

    \If{$rand() < p_{val}$} \Comment{Value mutation}
        \State $u \gets$ Child içinde seçili rastgele varlık
        \State $Child.S[u] \gets Child.S[u] \times (0.9 \text{ veya } 1.1)$
    \EndIf

    \State $(Child.Mask, Child.S) \gets \textsc{FixCardinality}(Child.Mask, Child.S, K)$

    \State $(f_{child}, \mathbf{w}_{child}) \gets \textsc{EvaluateWithRepair}(Child.Mask, Child.S, \lambda)$
    \State $WorstIdx \gets \arg\max(Fits)$
    \If{$f_{child} < Fits[WorstIdx]$}
        \State $P[WorstIdx] \gets Child$
        \State $Fits[WorstIdx] \gets f_{child}$
        \State $W[WorstIdx] \gets \mathbf{w}_{child}$
        \If{$f_{child} < f^*$}
            \State $\mathbf{w}^* \gets \mathbf{w}_{child}$, $f^* \gets f_{child}$
        \EndIf
    \EndIf
\EndFor

\State \Return $(\mathbf{w}^*, f^*)$
\end{algorithmic}
\end{algorithm}


\section{Deneysel Çalışma ve Bulgular}

Algoritma, OR-Library'deki 4 standart veri seti üzerinde test edilmiştir. Parametreler: $K=10$, $e_i=0.01$, $d_i=1.0$. Popülasyon büyüklüğü 100 ve iterasyon sayısı literatür standardı olan $1000 \times N$ olarak belirlenmiştir. Karşılaştırma için Gurobi 12.0.2 çözücüsü (MIQP) kullanılmıştır.

\subsection{Performans Metriği}
Çözüm kalitesi, Chang vd. \cite{chang2000} tarafından tanımlanan \emph{Ortalama Yüzde Hata (Mean Percentage Error)} metriği ile ölçülmüştür. Bu metrik, bulunan çözümün (risk, getiri) uzayında, kısıtsız etkin sınıra (UEF) olan hem dikey hem yatay uzaklığının minimumunu alarak hesaplanır.

\subsection{Sonuçların Karşılaştırılması}

Tablo \ref{tab:final_results_comprehensive}, önerilen Hibrit GA'nın sonuçlarını, Kesin Çözüm (MIQP) ve literatürdeki diğer önemli çalışmalarla (Chang-GA \cite{chang2000}, Deng-PSO \cite{deng2012}, Mansouri-ARO \cite{mansouri2021}) karşılaştırmaktadır.

\begin{table}[H]
\centering
\caption{Hibrit GA, MIQP ve Literatür Sonuçlarının Detaylı Karşılaştırılması (Amaç Fonksiyonu, Hata \% ve Süre)}
\label{tab:final_results_comprehensive}
\resizebox{\textwidth}{!}{
\begin{tabular}{lccccccccccccc}
\toprule
\textbf{Dataset} & \textbf{N} & \multicolumn{2}{c}{\textbf{Ortalama Amaç}} & \multicolumn{2}{c}{\textbf{Önerilen GA}} & \multicolumn{2}{c}{\textbf{MIQP (Exact)}} & \textbf{Gap} & \multicolumn{2}{c}{\textbf{Chang (GA)}} & \multicolumn{2}{c}{\textbf{Deng (PSO)}} & \textbf{Mansouri (ARO)} \\
\cmidrule(lr){3-4} \cmidrule(lr){5-6} \cmidrule(lr){7-8} \cmidrule(lr){10-11} \cmidrule(lr){12-13}
 & & \textbf{GA Obj} & \textbf{MIQP Obj} & \textbf{Hata (MPE) \%} & \textbf{Süre(s)} & \textbf{Hata (MPE) \%} & \textbf{Süre(s)} & \textbf{\%} & \textbf{Hata (MPE) \%} & \textbf{Süre(s)} & \textbf{Hata (MPE) \%} & \textbf{Süre(s)} & \textbf{Hata (MPE) \%} \\
\midrule
Port1 & 31 & -0.003900 & -0.003912 & 1.27 & 12.2 & 1.10 & 0.09 & 0.31 & 1.09 & 172 & 1.09 & 4.8 & 1.41 \\
Port2 & 85 & -0.004121 & -0.004149 & 2.88 & 26.0 & 2.33 & 4.13 & 2.50 & 2.54 & 544 & 2.54 & 26.8 & 1.31 \\
Port3 & 89 & -0.003490 & -0.003497 & 1.11 & 25.4 & 0.85 & 24.2 & 1.10 & 1.10 & 573 & 1.06 & 31.4 & 0.81 \\
Port4 & 98 & -0.003823 & -0.003837 & 1.94 & 27.9 & 1.37 & 334.2 & 2.05 & 1.93 & 638 & 1.68 & 34.5 & 1.44 \\
\bottomrule
\end{tabular}
}
\end{table}

\subsection{Tartışma}

Tablo \ref{tab:final_results_comprehensive} ile elde edilen sonuçlar, önerilen yaklaşımın (i) çözüm kalitesi, (ii) matematiksel amaç fonksiyona yakınlık ve (iii) hesaplama verimliliği açısından dengeli bir performans sunduğunu göstermektedir.

\textbf{1. Çözüm Kalitesi (MPE) ve Literatürle Uyum:}
Önerilen GA’nın MPE değerleri \%1.11–\%2.88 aralığında; MIQP’in MPE değerleri ise \%0.85–\%2.33 aralığında gözlenmiştir. Bu sonuçlar, GA’nın etkin sınıra yakın (geometrik olarak tutarlı) çözümler üretebildiğini göstermektedir. Ayrıca Port3 ve Port4 veri setlerinde önerilen GA’nın hata oranları (sırasıyla \%1.11 ve \%1.94), Chang vd.\ \cite{chang2000} tarafından raporlanan GA sonuçlarıyla (sırasıyla \%1.10 ve \%1.93) oldukça yakındır. Bu uyum, uygulamanın literatürdeki standart benchmarklarla tutarlı çalıştığını desteklemektedir.

\textbf{2. Amaç Fonksiyonu Yakınlığı (GA Obj vs MIQP Obj):}
Bu çalışmada literatürden farklı olarak, yalnızca MPE değil aynı zamanda amaç fonksiyonu değerleri de MIQP ile kıyaslanmıştır. Tablo \ref{tab:final_results_comprehensive} incelendiğinde GA–MIQP amaç değeri farkının göreli olarak düşük kaldığı (Gap \%0.31–\%2.50) görülmektedir. Örneğin Port4’te MIQP $-0.003837$ değerine ulaşırken önerilen GA $-0.003823$ elde etmiştir. Bu bulgu, GA’nın yalnızca etkin sınıra geometrik yakınlık sağlamadığını; aynı zamanda MIQP’in amaç fonksiyonu değerlerine de yakınsadığını göstermektedir. Bununla birlikte, GA’nın sezgisel doğası gereği bu yakınlık “kesin optimalite garantisi” olarak değil, “pratikte güçlü yakınsama göstergesi” olarak yorumlanmalıdır.

\textbf{3. Hesaplama Süresi ve Ölçek Etkisi:}
Süre sonuçları problem boyutuna ve çözücü türüne göre farklılaşmaktadır. Küçük/orta ölçekli örneklerde MIQP oldukça hızlı çalışabilmekte (ör.\ Port1), bazı örneklerde GA ile benzer sürelerde sonuç üretebilmektedir (ör.\ Port3). Buna karşın $N$’in 100’e yaklaştığı daha zor örneklerde (Port4, $N=98$) GA’nın belirgin bir hız avantajı görülmektedir: Port4’te GA 27.9 saniye, MIQP 334.2 saniye sürmüştür (yaklaşık 12 kat). Bu noktada adil yorum için, GA tarafında paralel $\lambda$ taraması kullanıldığı; MIQP tarafında ise çözücünün tek iş parçacığıyla (thread) çalıştırıldığı hatırlanmalıdır. Dolayısıyla süre kıyası, “yaklaşımın pratik karar desteği için uygunluğu” açısından anlamlıdır; ancak donanım/iş parçacığı ayarlarına duyarlıdır.

\textbf{4. Görünmez Bölgeler (Invisible Regions) ve Çözünürlük:}
Port1 veri setinde GA’nın hata oranı (\%1.27) MIQP’e (\%1.10) göre marjinal düzeyde daha yüksek olsa da amaç fonksiyonu değerleri birbirine çok yakındır ($-0.003900$ vs $-0.003912$; Gap \%0.31). Chang vd.\ \cite{chang2000}, CCEF’nin konveks olmayan yapısı nedeniyle, ağırlıklı toplam ($\lambda$) yöntemini kullanan kesin çözücülerin bazı “çukur” bölgelerdeki (invisible regions) çözümleri atlayabileceğini belirtmiştir. Popülasyon tabanlı yaklaşımlar ise çözüm uzayını daha geniş tarayabildiği için bu tür bölgelerde farklı ama rekabetçi çözümler üretebilmektedir. Bu nedenle Port1’deki küçük sapma, yöntemler arasındaki arama dinamiklerinin doğal bir sonucudur.

\section{Sonuç}

Bu projede, kardinalite kısıtlı portföy optimizasyonu problemi için onarım (repair) tabanlı ve \emph{steady-state} evrim şemasına sahip, yüksek performanslı bir Genetik Algoritma yaklaşımı uygulanmış ve sonuçlar MIQP (Gurobi) ile benchmark edilmiştir. OR-Library veri setleri (Port1–Port4) üzerinde yapılan deneyler doğrultusunda aşağıdaki çıkarımlar elde edilmiştir:

\begin{itemize}
    \item \textbf{Çözüm kalitesi:} Önerilen GA, MPE metriği açısından etkin sınıra yakın çözümler üretmiş; Port3 ve Port4 gibi örneklerde literatürde raporlanan Chang-GA \cite{chang2000} sonuçlarıyla çok yakın hata değerleri elde etmiştir.
    \item \textbf{Matematiksel hedefe yakınlık:} Amaç fonksiyonu değerleri bazında GA–MIQP farkının (Gap) genel olarak düşük kaldığı gözlenmiştir (Tablo \ref{tab:final_results_comprehensive}). Bu bulgu, GA’nın amaç fonksiyona göre de güçlü bir yakınsama sergilediğini göstermektedir.
    \item \textbf{Hesaplama verimliliği:} Süre performansı veri setine göre değişmektedir. Küçük örneklerde MIQP çok hızlı iken (ör.\ Port1), $N$’in 100’e yaklaştığı daha zor örneklerde (Port4, $N=98$) GA belirgin hız avantajı sağlamıştır (Port4’te yaklaşık 12 kat). Bu durum, yöntemin özellikle büyük/zor örneklerde pratik karar desteği açısından avantajlı olabileceğini göstermektedir.
    \item \textbf{Uygulama katkısı:} Numba hızlandırması ve paralel $\lambda$ taraması gibi modern programlama teknikleri, çözüm kalitesini korurken toplam çalışma süresini pratik düzeyde iyileştirmiştir.
\end{itemize}


\newpage

\begin{thebibliography}{99}

\bibitem{markowitz1952}
H.~Markowitz, 
``Portfolio Selection,'' 
\emph{The Journal of Finance}, 7(1), 77--91, 1952. 
URL: \url{https://doi.org/10.1111/j.1540-6261.1952.tb01525.x}

\bibitem{chang2000}
T.~J.~Chang, N.~Meade, J.~E.~Beasley, Y.~M.~Sharaiha, 
``Heuristics for cardinality constrained portfolio optimisation,'' 
\emph{Computers \& Operations Research}, 27(13), 1271--1302, 2000. 
URL: \url{https://doi.org/10.1016/S0305-0548(99)00074-X}

\bibitem{deng2012}
G.~F.~Deng, W.~T.~Lin, C.~C.~Lo, 
``Markowitz-based portfolio selection with cardinality constraints using improved particle swarm optimization,'' 
\emph{Expert Systems with Applications}, 39(4), 4558--4566, 2012. 
URL: \url{https://doi.org/10.1016/j.eswa.2011.09.129}

\bibitem{mansouri2021}
T.~Mansouri, M.~R.~S.~Moghadam, 
``Markowitz-based cardinality constrained portfolio selection using Asexual Reproduction Optimization (ARO),'' 
\emph{arXiv preprint arXiv:2101.03312}, 2021. 
URL: \url{https://arxiv.org/abs/2101.03312}

\bibitem{orlibrary}
J.~E.~Beasley, ``OR-Library: Portfolio Selection Problems,'' \url{https://people.brunel.ac.uk/~mastjjb/jeb/orlib/portinfo.html}.

\bibitem{kalayci2020}
C.~B.~Kalaycı, O.~Ertenlice, H.~Akyer, H.~Aygören,
``An efficient hybrid metaheuristic algorithm for cardinality constrained portfolio optimization,''
\emph{Swarm and Evolutionary Computation}, cilt 54, 100662, 2020.
URL: \url{https://www.sciencedirect.com/science/article/pii/S2210650218309702}

\bibitem{nikiporenko2023}
A.~Nikiporenko,
``Time-limited Metaheuristics for Cardinality-constrained Portfolio Optimisation,''
\emph{arXiv preprint}, arXiv:2307.04045, 2023.
URL: \url{https://arxiv.org/abs/2307.04045}

\bibitem{guarino2024}
A.~Guarino, D.~Santoro, L.~Grilli, R.~Zaccagnino, M.~Balbi,
``EvoFolio: a portfolio optimization method based on multi-objective evolutionary algorithms,''
\emph{Neural Computing and Applications}, cilt 36, ss.\ 7221--7243, 2024.
URL: \url{https://link.springer.com/article/10.1007/s00521-024-09456-w}

\bibitem{mwamba2025}
J.~W.~M.~Muteba Mwamba, L.~M.~Mbucici,
``Optimizing the Complexity of Higher-Order Moment Portfolio Optimization Using NSGA-III,''
\emph{Preprints}, 2024 (basım öncesi sürüm).
URL: \url{https://www.preprints.org/manuscript/202409.1019/v1}

\end{thebibliography}

\end{document}
