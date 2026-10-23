O algoritmo de Rabin-Karp para busca em textos
==============================================

Do Ctrl+F ao problema
---------------------

Abra um PDF de 300 páginas, aperte [[Ctrl]] + [[F]] e digite uma palavra. Em uma fração de segundo, o programa mostra todos os lugares onde ela aparece. Parece trivial, mas alguém precisou decidir *como* procurar, e essa decisão pesa quando o texto tem milhões de caracteres: um livro, o código de um projeto inteiro, um genoma.

Vamos dar nomes às coisas. A **entrada** é:

* um **texto** $T$ com $n$ caracteres;

* um **padrão** $P$ com $m$ caracteres, sendo $m \leq n$.

A **saída** é a lista de todas as posições de $T$ onde começa uma cópia exata de $P$.

Para dar para fazer na mão, vamos usar um exemplo minúsculo ao longo de todo o handout. As posições são contadas a partir de 0, como nos vetores de C.

| posição | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---------|---|---|---|---|---|---|---|---|---|---|----|
| $T$     | C | A | D | A | E | D | B | H | D | A | E  |

Aqui $n = 11$ e o padrão é `DAE`, com $m = 3$.

A solução ingênua: força bruta
------------------------------

Antes de inventar qualquer coisa esperta, vale entender muito bem a solução mais simples. Ela vai ser o ponto de comparação para tudo que vem depois.


??? Atividade 1: procurando na mão

Ache todas as ocorrências de `DAE` em `CADAEDBHDAE` do jeito mais direto que você imaginar, como se fosse com lápis e papel. Enquanto procura, conte quantas vezes você compara **uma letra do texto com uma letra do padrão**. Se uma posição já mostrou que não pode ser uma ocorrência, desista dela na hora, sem olhar o resto.

1. Em quais posições `DAE` aparece?

2. Quantas comparações de letras você fez no total?

::: Gabarito

1. Nas posições 2 e 8.

2. Foram 14 comparações:

| posição | trecho de $T$ | comparações | o que aconteceu |
|---------|---------------|-------------|-----------------|
| 0       | CAD           | 1           | C ≠ D, desiste  |
| 1       | ADA           | 1           | A ≠ D, desiste  |
| 2       | DAE           | 3           | D = D, A = A, E = E: **ocorrência** |
| 3       | AED           | 1           | A ≠ D, desiste  |
| 4       | EDB           | 1           | E ≠ D, desiste  |
| 5       | DBH           | 2           | D = D, B ≠ A, desiste |
| 6       | BHD           | 1           | B ≠ D, desiste  |
| 7       | HDA           | 1           | H ≠ D, desiste  |
| 8       | DAE           | 3           | D = D, A = A, E = E: **ocorrência** |

Total: $1 + 1 + 3 + 1 + 1 + 2 + 1 + 1 + 3 = 14$.

:::

???

O que você acabou de fazer tem nome: **força bruta** (ou busca ingênua). Imagine uma **janela** de tamanho $m$ posicionada sobre o texto, começando na posição 0. Em cada posição, compare a janela com o padrão, letra por letra, da esquerda para a direita, e **pare no primeiro par de letras diferentes**. Se chegar ao fim sem diferença, anote a posição. Depois, deslize a janela uma posição para a direita e repita, até a janela encostar no final do texto.

Use as setas abaixo para acompanhar a execução no nosso exemplo. O `i` marca onde a janela começa, o `j` marca a letra do padrão que está sendo comparada e o sinal entre as duas linhas mostra o resultado da comparação. Em vermelho ficam as letras que já casaram.

:forca-bruta

Em C, com o texto em `T` (tamanho `n`) e o padrão em `P` (tamanho `m`), fica assim:

``` c
void forca_bruta(char *T, int n, char *P, int m) {
    for (int i = 0; i + m <= n; i++) {
        int j = 0;
        while (j < m && T[i + j] == P[j]) {
            j++;
        }
        if (j == m) {
            printf("achei em %d\n", i);
        }
    }
}
```

O `for` escolhe a posição da janela. O `while` compara letra por letra e para no primeiro erro. No final, `~j == m` só acontece se **todas** as letras bateram.


??? Atividade 2: o pior que pode acontecer

O custo da força bruta depende do conteúdo do texto e do padrão, não só dos tamanhos. Considere $n = 12$ e $m = 3$.

1. Quantas janelas o algoritmo examina?

2. No máximo, quantas comparações ele faz em uma única janela?

3. Escreva um texto de 12 letras e um padrão de 3 letras que obriguem o algoritmo a fazer esse máximo **em todas as janelas**.

,,, Dica 1

Pense numa janela de cada vez: para o algoritmo fazer o máximo de comparações, em que momento ele pode desistir?

,,,

,,, Dica 2

Só dá para chegar na última letra da janela se todas as anteriores forem iguais às do padrão. Que letras repetidas você poderia usar?

,,,

::: Gabarito

1. $n - m + 1 = 10$ janelas, nas posições 0 a 9.

2. $m = 3$ comparações, que acontecem quando o algoritmo só desiste na última letra da janela ou nem chega a desistir.

3. Por exemplo, o texto `AAAAAAAAAAAA` com o padrão `AAB`: em toda janela as duas primeiras letras casam e só a terceira diferencia, então são 3 comparações por janela. Outra opção é o mesmo texto com o padrão `AAA`, em que toda janela casa por inteiro. Nos dois casos, $3 \cdot 10 = 30$ comparações.

:::

???

Em geral, são $n - m + 1$ janelas e até $m$ comparações em cada uma. No pior caso, são $m \cdot (n - m + 1)$ comparações, o que é da ordem de $n \cdot m$. Escrevemos $O(n \cdot m)$. Se o padrão cresce junto com o texto (por exemplo, $m = n / 2$), isso é quadrático.

O gargalo está claro: cada janela pode custar até $m$ comparações letra por letra, e nada nesse algoritmo evita isso. Daqui em diante, o objetivo é atacar exatamente esse custo.

A ideia: comparar impressões digitais
-------------------------------------

Cada janela custa até $m$ comparações porque comparamos letra por letra. E se pudéssemos resumir uma palavra inteira em **um único número inteiro**, uma espécie de impressão digital? Se palavras iguais tiverem o mesmo número, em vez de comparar a janela com o padrão letra por letra, compararíamos **dois inteiros**.


!!! Por que isso acelera

Comparar `DAE` com `DAE` letra por letra custa até 3 comparações. Comparar os dois inteiros que representam essas palavras custa 1, e continuaria custando 1 mesmo que as palavras tivessem 3 mil letras. Essa é toda a graça da ideia: o custo da comparação deixa de depender do tamanho da palavra.

Isso vale desde que o número caiba numa variável. Guarde essa condição, ela vai dar trabalho mais adiante.

!!!


??? Atividade 3: como transformar uma palavra em número?

Para o nosso exemplo, só precisamos destas letras:

| letra | A | B | C | D | E | H |
|-------|---|---|---|---|---|---|
| valor | 1 | 2 | 3 | 4 | 5 | 8 |

Primeira tentativa: a impressão digital de uma palavra é a **soma** dos valores das suas letras. Assim, `DAE` vira $4 + 1 + 5 = 10$.

1. Ache uma palavra de 3 letras, diferente de `DAE`, com a mesma impressão digital.

2. Por que isso atrapalha o plano de comparar só os inteiros?

3. O que a soma está ignorando? Como você faria a ordem das letras importar?

,,, Dica

Os números 415 e 154 têm os mesmos dígitos. O que faz deles números diferentes?

,,,

::: Gabarito

1. Qualquer reordenação serve: `EAD`, `ADE`, `AED`, `DEA` ou `EDA`, todas somam 10. Nem precisa ser reordenação: `AHA` também soma $1 + 8 + 1 = 10$.

2. Se a impressão digital de uma janela for 10, não dá para concluir que a janela é `DAE`: números iguais não garantem palavras iguais. Isso é uma **colisão**, e ela estraga a ideia de comparar só os inteiros.

3. A soma ignora a **posição** das letras. Para a ordem importar, cada posição precisa ter um peso diferente, como nos números decimais: em 415, o 4 vale 400, o 1 vale 10 e o 5 vale 5.

:::

???

É exatamente a ideia de usar a posição como peso. Tratamos a palavra como um número escrito em base 10, em que cada letra é um dígito:

$$\text{DAE} \rightarrow 4 \cdot 10^2 + 1 \cdot 10^1 + 5 \cdot 10^0 = 415$$

Agora `EAD` vira $5 \cdot 10^2 + 1 \cdot 10^1 + 4 \cdot 10^0 = 514$, que é diferente de 415. Como todo valor é menor que 10, cada número corresponde a **uma única** palavra de 3 letras: dois textos têm o mesmo inteiro se e somente se são iguais. Ou seja, comparar inteiros passa a ser tão confiável quanto comparar letra por letra.


!!! Duas perguntas em aberto

Parece que resolvemos o problema, mas o plano tem duas furadas:

1. E se a palavra tiver 25 letras? O número terá 25 dígitos, que não cabe nem em um `long long` (que vai até cerca de $9 \cdot 10^{18}$, ou seja, 19 dígitos). Aí comparar dois inteiros volta a custar proporcional ao tamanho da palavra.

2. Mesmo que o número caiba: como calculamos o inteiro de **cada janela do texto**? Se cada um exigir olhar as $m$ letras da janela, voltamos ao custo da força bruta.

As próximas seções atacam essas duas furadas, nessa ordem.

!!!
