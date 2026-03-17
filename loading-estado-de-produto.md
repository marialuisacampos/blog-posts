---
title: "Pare de tratar loading como estado de React — ele é estado de produto"
date: 2026-03-17
excerpt: Loading não é só um booleano de renderização no React. Neste artigo, exploro por que ele deve ser tratado como um estado de produto, conectando Suspense, transitions, estabilidade visual, skeletons, indicadores de progresso e acessibilidade para construir experiências de espera mais confiáveis e menos bruscas.
tags:
  [frontend, react, ux, performance, web-performance, suspense, accessibility, product]
---

# Pare de tratar loading como estado de React — ele é estado de produto

Em muito código React, loading ainda aparece como uma decisão local de componente: `if (isLoading) return <Spinner />`. O problema é que, num produto real, loading nunca é só um booleano de renderização. Ele define o que continua visível, o que some, o que pode ser usado, se a interface parece estável e se a espera transmite confiança ou bagunça.

Dito de outra forma: loading faz parte da experiência tanto quanto a tela final. Se o usuário perde contexto, vê a interface piscar, sofre com layout shift ou não entende se o sistema ainda está processando, então existe um problema de produto entregue pela UI.

## Uma introdução curta ao Suspense

O `Suspense` existe porque React reconhece que a espera precisa ser modelada com mais intenção. Ele funciona como uma boundary de loading: você define um `fallback`, e o React mostra esse fallback enquanto a parte da árvore dentro daquela boundary ainda não pode renderizar. Quando o conteúdo fica pronto, substitui o fallback. Você pode aninhar boundaries para revelar partes da tela em etapas, em vez de travar tudo atrás de um único loading.

Importante: o React não “vê” qualquer fetch como algo que deva acionar o fallback do Suspense. Ou seja: o Suspense não observa qualquer operação assíncrona arbitrária; ele só entra em ação quando a renderização encontra uma fonte de dados que foi integrada ao modelo de suspensão do React.

Isso importa porque evita um erro de raciocínio comum: achar que "usar Suspense" resolve sozinho a experiência de loading. Ele ajuda a declarar boundaries de espera, mas a qualidade da experiência ainda depende de como você desenha o fallback, como organiza a árvore, se preserva contexto visual e se escolhe o tipo certo de indicador para cada situação.

## O erro mais comum: loading como troca binária de tela

O modelo mental mais pobre para loading é esse: ou mostra o conteúdo, ou mostra um spinner. Na prática, isso tende a gerar experiências bruscas. A própria documentação do React alerta para isso: quando um componente suspende, a boundary mais próxima troca para o `fallback`, e isso pode criar uma experiência "jarring" se aquela área já estava mostrando conteúdo útil.

Se o usuário já estava vendo título, filtros, navegação ou resultados anteriores, esconder tudo só porque uma nova busca começou destrói contexto visual. O ideal é preservar o que já foi revelado e introduzir a próxima atualização de forma menos abrupta.

## O que React já oferece para lidar com isso

`startTransition` e `useTransition` existem justamente para marcar certas atualizações como não bloqueantes.

Não estou aqui dizendo que devemos usar transition em tudo, mas sim para te fazer entender o princípio por trás dessas APIs: nem toda atualização deveria resetar a percepção da tela.

Às vezes, manter a interface anterior visível (talvez com um estado intermediário mais sutil) é muito melhor do que substituir tudo por um spinner centralizado.

Pode parecer algo simples, mas muda completamente a experiência do usuário.

## Loading ruim também vira problema de performance percebida

O usuário percebe mais do que o tempo bruto de espera. Ele percebe se a interface se move, se o layout pula, se o botão sai do lugar, se a lista cresce empurrando o conteúdo. Isso é estabilidade visual, medida pelo CLS (métrica importante de performance web), com recomendação de 0,1 ou menos no percentil 75 das visitas.

A conexão com loading é direta: entre as causas mais comuns de CLS ruim estão conteúdo injetado dinamicamente sem espaço reservado e elementos que entram depois empurrando o que já estava na tela. Um fallback mal planejado ou um skeleton que não respeita a estrutura real da interface gera instabilidade visual mensurável (E É FEIO).

Por isso, uma boa prática não é só "mostrar alguma coisa enquanto espera". É reservar espaço de forma coerente com o conteúdo que vai chegar. Se você sabe que um card terá imagem, título e metadados, o skeleton deveria refletir essa estrutura com dimensões próximas.

## Skeleton não é maquiagem: ele precisa preservar estrutura

O skeleton funciona quando comunica forma, hierarquia e expectativa. Skeleton screens são placeholders que dão pistas de como a página ficará quando estiver pronta, reduzindo a sensação de espera porque o usuário já enxerga a estrutura.

Mas isso só funciona quando o placeholder é fiel ao layout real. Um skeleton genérico com blocos aleatórios pode manter a tela "ocupada", mas falha em dois pontos: não informa o que está chegando E cria uma troca visual brusca quando o conteúdo final entra. 

## Spinner, skeleton ou barra de progresso?

Nem toda espera pede o mesmo feedback. Para processos curtos, um spinner pode bastar; para páginas ou áreas estruturadas, skeleton tende a funcionar melhor porque mostra a forma do conteúdo; para esperas acima de 10 segundos, a recomendação é sair da animação vaga e usar uma barra de progresso ou estimativa explícita de duração.

Indo mais além, inline loading é indicado para ações curtas e localizadas, enquanto áreas maiores se beneficiam mais de skeleton states. O design system também recomenda evitar inline loading para múltiplos itens ao mesmo tempo e, em data tables, prefere skeleton a spinner quando há espera perceptível.

Uma régua simples: spinner para esperas curtas e pontuais, skeleton para estrutura previsível, progress bar para espera longa com noção de avanço. O erro é usar o mesmo componente de loading para qualquer cenário, pois isso trata esperas diferentes como se fossem equivalentes, e o usuário sente a diferença.

## Loading progressivo é melhor do que loading binário

Nested boundaries do `Suspense` permitem que partes diferentes da interface apareçam em sequência, em vez de exigir que tudo esteja pronto ao mesmo tempo. Isso melhora a percepção de velocidade porque a tela começa a ganhar utilidade antes do estado final completo.

Algo legal a se frizar é que esse modelo fica ainda mais interessante com streaming SSR. A documentação de `renderToReadableStream` explica que, quando parte da árvore está dentro de uma `Suspense` boundary, o React pode enviar primeiro o HTML do fallback e continuar revelando mais conteúdo conforme os dados ficam disponíveis. "Carregar" não precisa ser uma única fase opaca. Na verdade, pode ser um processo progressivo, com entrega incremental.

## Loading também precisa ser acessível

Há um erro silencioso em muitos produtos: loading visual sem feedback acessível. O MDN recomenda usar `aria-busy="true"` quando uma região está sendo atualizada e só voltar para `false` quando a atualização terminar, para evitar que tecnologias assistivas anunciem mudanças antes da hora. Live regions existem justamente para expor atualizações dinâmicas de conteúdo que acontecem depois do carregamento inicial.

Loading não afeta só quem vê o spinner. Se a interface atualiza conteúdo de forma dinâmica, o sistema precisa comunicar isso também para leitores de tela. Funcionar visualmente não basta.

## O que evitar

Blank screen enquanto tudo carrega. Spinner de página inteira para interações pequenas. Substituir conteúdo já útil por fallback a cada refetch. Skeleton sem preservar estrutura real. Barra de progresso que não representa progresso de verdade. Todos esses padrões pioram confiança, contexto e percepção de velocidade (e aparecem o tempo todo).

Também vale não transformar loading em maquiagem para arquitetura ruim. Se sua busca depende de `useEffect` manual em cascata, se toda mudança de filtro reinicializa a tela inteira, trocar o spinner por um skeleton bonito não resolve a raiz. Antes de tudo, você precisa ter seu código otimizado.

## Um checklist para pensar loading melhor

Antes de implementar o próximo `isLoading`, vale passar por estas perguntas:

- O que já está na tela precisa realmente sumir?
- O fallback preserva espaço suficiente para evitar layout shift?
- O tipo de loading combina com a duração e a natureza da espera?
- Eu consigo revelar partes da UI em etapas com boundaries menores?
- Essa atualização está acessível para tecnologias assistivas?
- Estou resolvendo um problema de loading ou escondendo um problema de arquitetura de dados?

## Por fim...

React oferece ferramentas boas para isso — `Suspense`, nested boundaries, transitions. Mas elas não decidem sozinhas o que manter visível, quando usar skeleton, quando usar progress bar ou como comunicar a atualização com acessibilidade. Essas são decisões de produto, e tratá-las como detalhes de renderização é o que transforma uma espera funcional numa experiência ruim.
