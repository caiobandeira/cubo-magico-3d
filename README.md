# Cubo Mágico 3D — Three.js

Projeto da disciplina de Computação Gráfica.

**Integrantes:** Caio de Araújo e Guilherme Marques

Simulação interativa de um cubo mágico (Rubik's Cube) feita em Three.js, com
câmera orbital e animação suave ao girar as faces.

## Como executar

Abra o `index.html` em qualquer navegador. O Three.js e o OrbitControls são
carregados por CDN, então é preciso estar com internet na primeira vez que a
página abrir.

## Controles

| Tecla / mouse | O que faz |
|---|---|
| `U` | Gira a face de cima (Up) |
| `D` | Gira a face de baixo (Down) |
| `R` | Gira a face da direita (Right) |
| `L` | Gira a face da esquerda (Left) |
| `F` | Gira a face da frente (Front) |
| `B` | Gira a face de trás (Back) |
| `Shift` + tecla | Gira a face no sentido contrário |
| Mouse (arrastar) | Orbita a câmera em volta do cubo |
| Mouse (roda) | Aproxima e afasta a câmera |
| Botão Embaralhar | Sorteia 20 movimentos e executa todos animados |
| Botão Resetar | Volta o cubo para o estado resolvido |

## O que foi implementado

**Requisitos mínimos**

| Requisito | Onde está | Linha |
|---|---|---|
| Cubo 3×3×3 com 27 cubinhos coloridos | Três laços encaixados criando as 27 combinações | 145 |
| Rotação de pelo menos 2 faces | As 6 faces giram (`U D R L F B`) | 178, 210 |
| Animação suave (interpolação do ângulo) | Smoothstep aplicado ao ângulo no loop | 350 |
| Câmera órbita com mouse (OrbitControls) | `new THREE.OrbitControls(...)` | 99 |
| Cores corretas por face | Objeto `CORES` e a montagem dos materiais | 118, 145 |

**Extras**

| Extra | Onde está | Linha |
|---|---|---|
| Todas as 6 faces rotacionáveis | Tabela `MOVIMENTOS` com as 6 letras | 178 |
| Botão "embaralhar" com animação | Função `embaralhar` + fila processada no loop | 268 |
| Contador de movimentos | Variável `contador`, atualizada em `girarFace` | 210 |

---

# Explicação do código

## 1. Os três pilares

Igual aos exemplos vistos em aula: `Scene` (o mundo), `PerspectiveCamera` (o
ponto de vista) e `WebGLRenderer` (quem desenha). A câmera fica em (5, 5, 7),
numa diagonal que deixa três faces visíveis ao mesmo tempo.

O `OrbitControls` é a única coisa nova: ele pega os eventos de mouse e move a
câmera em volta do ponto (0,0,0), que é justamente o centro do cubo.

```js
const controls = new THREE.OrbitControls(camera, renderer.domElement);
controls.enableDamping = true;  // movimento mais suave
controls.minDistance = 6;       // não deixa a câmera entrar dentro do cubo
controls.maxDistance = 20;
```

Ele precisa de `controls.update()` dentro do loop de animação pra funcionar.

## 2. Como o cubo é montado (linha 145)

O cubo não é um objeto só: são **27 cubinhos independentes**. Três laços
encaixados varrem x, y e z de −1 a 1, o que dá exatamente 27 combinações:

```js
for (let x = -1; x <= 1; x++) {
  for (let y = -1; y <= 1; y++) {
    for (let z = -1; z <= 1; z++) {
      // cria um cubinho na posição (x, y, z)
    }
  }
}
```

A posição de cada cubinho é um número inteiro (−1, 0 ou 1). Isso é importante
e volta mais pra frente: é assim que o código descobre quais cubinhos fazem
parte de cada face.

### As cores

Um `BoxGeometry` aceita **um material por face**, e a ordem é sempre essa:

```
[0] = +X (direita)   [1] = -X (esquerda)
[2] = +Y (cima)      [3] = -Y (baixo)
[4] = +Z (frente)    [5] = -Z (trás)
```

Então o código monta um array de 6 materiais e pinta cada face só se o cubinho
estiver naquela camada externa:

```js
new THREE.MeshBasicMaterial({ color: x === 1 ? CORES.direita : PLASTICO })
```

Lendo em português: "se esse cubinho está na camada x = 1, a face da direita
dele é vermelha; se não, é preta". As faces internas ficam pretas porque elas
nunca aparecem — e é justamente o preto nas frestas que dá o visual de cubo
mágico de verdade.

O `BoxGeometry(0.95, 0.95, 0.95)` com os cubinhos espaçados de 1 em 1 deixa uma
folga de 0,05 entre eles. É essa folga que forma as linhas pretas.

## 3. Como uma face gira — a parte principal

Esse é o problema central do projeto, e a dica do enunciado é exatamente a
solução usada aqui.

O problema: girar uma face significa girar 9 cubinhos **juntos, em volta do
centro do cubo**. Se cada cubinho girasse em torno do próprio centro, eles
girariam no lugar em vez de rodar em volta do eixo.

A solução é usar um `THREE.Group` como pivô — é o mesmo conceito do
`ctx.save()` / `ctx.restore()` do Canvas 2D e dos grupos do exemplo do sistema
solar: **transformar o pai transforma todos os filhos**.

São três etapas:

### Etapa 1 — recrutar os cubinhos (linha 210)

```js
for (const c of cubinhos) {
  if (Math.round(c.position[mov.eixo]) === mov.camada) {
    pivo.attach(c);
  }
}
```

Como cada movimento guarda um eixo e uma camada, a comparação acima pega
exatamente os 9 cubinhos da face. Por exemplo, o movimento `U` tem
`eixo: "y"` e `camada: 1`, então ele seleciona todos os cubinhos com y = 1.

O ponto delicado é o **`attach` em vez de `add`**. Os dois mudam o pai do
objeto, mas:

- `add` mantém a posição **local** — o objeto pula para outro lugar na tela
- `attach` mantém a posição **no mundo** — o objeto continua onde está

Como o pivô está na origem e sem rotação nesse instante, aqui os dois dariam o
mesmo resultado. Mas na volta (etapa 3) a diferença é crucial, então o código
usa `attach` nos dois lados por coerência.

### Etapa 2 — animar o giro (linha 338)

O giro não acontece de uma vez. O loop de animação vai aumentando uma variável
`t` de 0 até 1, e o ângulo do pivô é calculado a partir dela:

```js
t += VELOCIDADE;
const suave = t * t * (3 - 2 * t);
pivo.rotation[eixoAtual] = anguloAlvo * suave;
```

A fórmula `t * t * (3 - 2t)` é a **smoothstep**. Ela transforma um `t` que
cresce de forma constante numa curva que começa devagar, acelera no meio e
freia no fim. É isso que o enunciado chama de "interpolação do ângulo" — sem
ela o giro ficaria mecânico, parando de repente.

Vale notar que `pivo.rotation[eixoAtual]` usa colchetes porque `eixoAtual` é
uma string (`"x"`, `"y"` ou `"z"`). Isso evita ter que escrever um `if` pra
cada eixo.

### Etapa 3 — devolver os cubinhos (linha 246)

Quando `t` chega em 1, o giro acabou e os cubinhos precisam sair do pivô:

```js
const filhos = pivo.children.slice();

for (const c of filhos) {
  cubo.attach(c);
  c.position.x = Math.round(c.position.x);
  c.position.y = Math.round(c.position.y);
  c.position.z = Math.round(c.position.z);
}

pivo.rotation.set(0, 0, 0);
```

Três detalhes que valem explicar:

**O `.slice()`** cria uma cópia da lista de filhos. Sem isso o laço quebraria,
porque `attach` remove o cubinho de `pivo.children` enquanto a gente está
percorrendo esse mesmo array.

**O `attach`** aqui é o que importa de verdade: o pivô está girado 90°, e os
cubinhos precisam continuar exatamente onde aparecem na tela depois de voltar
pro grupo do cubo. O `attach` recalcula a posição e a rotação local pra que o
resultado visual não mude. Com `add` o cubo se desmontaria.

**O arredondamento** conserta o acúmulo de erro de ponto flutuante. Depois de
vários giros, uma posição que deveria ser 1 vira algo como 0.9999997, e a
comparação `=== 1` da etapa 1 pararia de encontrar os cubinhos. Arredondar a
cada giro mantém tudo inteiro.

Por fim, `pivo.rotation.set(0, 0, 0)` zera o pivô pra ele estar limpo no
próximo movimento — o mesmo pivô é reaproveitado o tempo todo.

## 4. A tabela de movimentos (linha 178)

```js
const MOVIMENTOS = {
  U: { eixo: "y", camada:  1, sentido: -1 },
  D: { eixo: "y", camada: -1, sentido:  1 },
  R: { eixo: "x", camada:  1, sentido: -1 },
  L: { eixo: "x", camada: -1, sentido:  1 },
  F: { eixo: "z", camada:  1, sentido: -1 },
  B: { eixo: "z", camada: -1, sentido:  1 }
};
```

Em vez de escrever seis funções parecidas, o código descreve os seis
movimentos como dados e tem **uma única função** que sabe girar qualquer um
deles. Foi isso que deixou barato implementar o extra das 6 faces: a tabela
cresceu, a lógica não.

O `sentido` serve pra que o giro saia no sentido horário quando se olha de
frente pra face. Faces opostas têm sinais opostos porque são vistas de lados
contrários — a face de cima e a de baixo giram em torno do mesmo eixo Y, mas
"horário" para uma é "anti-horário" para a outra.

## 5. Embaralhar e o sistema de fila (linha 268)

Só pode haver um giro acontecendo por vez — a trava é a variável `girando`. Se
o embaralhar simplesmente chamasse `girarFace` 20 vezes seguidas, 19 chamadas
seriam ignoradas.

A solução é uma **fila**. O embaralhar só enfileira:

```js
for (let i = 0; i < 20; i++) {
  fila.push({
    nome: letras[Math.floor(Math.random() * 6)],
    invertido: Math.random() < 0.5
  });
}
```

E o loop de animação puxa um movimento de cada vez, sempre que estiver livre:

```js
if (!girando && fila.length > 0) {
  const prox = fila.shift();
  girarFace(prox.nome, prox.invertido, false);
}
```

O `false` no fim é o parâmetro `conta`: os movimentos do embaralhar não entram
no contador. Quando a fila esvazia, o contador é zerado pra pessoa começar a
resolver do zero.

## 6. Resetar (linha 287)

Como os cubinhos foram criados na ordem dos três laços, dá pra reconstruir a
posição original de cada um percorrendo os mesmos laços na mesma ordem:

```js
c.position.set(x, y, z);     // posição original
c.rotation.set(0, 0, 0);     // apaga as rotações acumuladas
```

O `rotation.set(0, 0, 0)` é necessário porque cada giro deixa uma rotação
guardada no cubinho. Sem isso o cubo voltaria para o lugar certo, mas com
cubinhos virados e as cores trocadas.

---

# Dificuldades encontradas

- Entender a diferença entre `add` e `attach`. Com `add` o cubo se desmontava
  ao terminar o giro, porque o cubinho mantinha a posição local e era jogado de
  volta para o lugar antigo.
- O acúmulo de erro de ponto flutuante. Depois de uns 10 movimentos as faces
  paravam de girar, porque as posições não eram mais exatamente −1, 0 e 1 e a
  comparação não encontrava mais os cubinhos.
- Fazer o embaralhar animado. A primeira versão chamava a rotação 20 vezes
  seguidas e só o primeiro movimento acontecia, por causa da trava `girando`.
  Resolvido com a fila.
- Acertar o sinal do `sentido` de cada face, pra que o giro saísse no sentido
  horário olhando de frente para cada uma delas.

---

# Perguntas frequentes sobre o código

**Por que usar um Group em vez de girar cada cubinho?**
Porque a face gira em volta do centro do cubo, não em volta do centro de cada
cubinho. Colocando os 9 dentro de um grupo e girando o grupo, cada um descreve
o arco correto automaticamente — é a mesma ideia da Lua orbitando a Terra no
exemplo do sistema solar.

**Qual a diferença entre `add` e `attach`?**
`add` troca o pai mantendo a posição local, então o objeto salta na tela.
`attach` troca o pai mantendo a posição no mundo, recalculando a posição local
pra compensar. Quando os cubinhos saem do pivô girado, só o `attach` preserva
o resultado visual.

**Por que arredondar as posições?**
Porque as contas em ponto flutuante acumulam erro a cada giro. Sem o
arredondamento, depois de alguns movimentos a posição vira 0.9999997 e a
seleção dos cubinhos da face para de funcionar.

**Como funciona a animação suave?**
Uma variável `t` vai de 0 a 1 ao longo do giro. O ângulo aplicado não é
`anguloAlvo * t`, e sim `anguloAlvo * (t·t·(3−2t))`. Essa curva faz o giro
começar devagar, acelerar e frear.

**Por que `MeshBasicMaterial` e não um material com iluminação?**
Porque iluminação não foi vista em aula ainda, e o cubo mágico não precisa:
as cores são chapadas mesmo, e o que separa visualmente uma face da outra são
as frestas pretas entre os cubinhos.

**Onde está a hierarquia de transformações?**
Na estrutura `scene → cubo → pivô → cubinhos`. Durante o giro os 9 cubinhos da
face são filhos do pivô, e a rotação do pivô se propaga para todos eles.
