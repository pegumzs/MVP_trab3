Sistema de suporte à decisão para triagem de mensagens suspeitas

MVP da disciplina Sistemas de Suporte à Decisão Departamento de Engenharia de Produção, Universidade de Brasília Professor: André Luiz Marques Serrano Aluno(a): Pedro Augusto de Menezes 

Este projeto classifica mensagens de texto como golpe ou legítima. A partir da probabilidade de golpe, ele recomenda uma ação: liberar a mensagem, mandar para revisão da equipe de TI ou bloquear. Todo o pipeline, da coleta à análise, está no notebook Projeto_SSD_MVP_Triagem_de_Mensagens.ipynb.

Sumário
Objetivo
Como executar
Base de dados e coleta
Modelagem
Qualidade dos dados
Transformação e carga
Resultados
Discussão geral
Ética, licença e proteção de dados
Autoavaliação
Referências
1. Objetivo
Problema

Golpes por mensagem de texto, conhecidos como smishing, tentam convencer a pessoa a transferir dinheiro ou passar dados. Em uma empresa, cada golpe que chega a um funcionário é um risco, e revisar todas as mensagens à mão não é viável.

O problema é decidir, com pouca intervenção humana, o que fazer com cada mensagem recebida: liberar, encaminhar para revisão ou bloquear.

Hipótese

Um classificador simples, treinado com mensagens já rotuladas, consegue, no conjunto de teste:

bloquear pelo menos 90% dos golpes;
bloquear no máximo 5% das mensagens legítimas;
mandar no máximo 10% das mensagens para revisão humana.
Perguntas de negócio
#	Pergunta	Decisão que a resposta apoia
P1	Que características do texto diferenciam os golpes das mensagens legítimas?	Quais sinais usar para orientar os funcionários e criar regras de filtro
P2	Entre a árvore de decisão e o Naive Bayes, qual modelo deixa passar menos golpes, e quantos alarmes falsos cada um gera?	Qual modelo adotar
P3	Com a regra de decisão proposta, quantos golpes são bloqueados, quantas mensagens vão para revisão e quantas legítimas são bloqueadas?	Se o sistema pode ser adotado e quanto trabalho ele gera para a TI
P4	Como a escolha do limite de bloqueio muda o número de golpes não bloqueados e de legítimas bloqueadas?	Qual limite a empresa deve usar
P5	O modelo treinado com mensagens de Moçambique reconhece golpes comuns no Brasil?	Se o modelo pode ser usado no Brasil sem novo treinamento
2. Como executar
Abra o Google Colab e vá em Arquivo > Fazer upload de notebook.
Escolha o arquivo Projeto_SSD_MVP_Triagem_de_Mensagens.ipynb.
Rode tudo em Ambiente de execução > Executar tudo.

O notebook baixa a base sozinho. Todas as bibliotecas usadas já vêm instaladas no Colab: pandas, numpy, matplotlib, seaborn, scikit-learn e nltk.

Por padrão, as camadas de dados ficam salvas em /content/mvp_smishing e somem quando a sessão do Colab termina. Para guardar no Google Drive, troque usar_google_drive para True na seção 2 do notebook e autorize o acesso quando o Colab pedir.

3. Base de dados e coleta

Procurei uma base pública de mensagens de golpe em português, com rótulo indicando se cada mensagem é golpe ou não, e não encontrei uma base brasileira assim.

A MOZ-Smishing foi a opção mais próxima:

tem mensagens reais em português;
as mensagens são rotuladas;
foi publicada junto com um artigo científico.

As mensagens vêm de Moçambique. Essa diferença de contexto é avaliada na pergunta P5.

Item	Valor
Nome	MOZ-Smishing
Origem	https://huggingface.co/datasets/MOZNLP/MOZ-Smishing
Licença	CreativeML OpenRAIL-M, conforme a página da base
Formato original	Um arquivo CSV (test.csv) com as colunas id, source, text e label
Técnica de coleta	Download direto do arquivo, sem web scraping
Data da coleta	28/09/2026
Tamanho do arquivo	315,3 kB
Volume	2.561 mensagens: 2.009 legítimas e 552 golpes
4. Modelagem

O modelo segue a organização de Data Lake, com uma tabela plana por conceito em cada camada.

O esquema estrela não foi usado porque o projeto tem um único conceito central, a mensagem. Nenhum atributo justifica uma tabela de dimensão separada.

Camada	Arquivo	Uma linha representa
bronze	bronze/moz_smishing.csv	Uma linha do arquivo original, sem alteração
silver	silver/mensagens.csv	Uma mensagem única, tratada e anonimizada
gold	gold/sinais_por_classe.csv	Uma classe (golpe ou legítima)
gold	gold/predicoes.csv	Uma mensagem do conjunto de teste, com risco e ação recomendada
gold	gold/simulacao_limites.csv	Um valor de limite de bloqueio

A linhagem dos dados é:

arquivo do Hugging Face;
camada bronze;
regras R1 a R8;
camada silver;
análises;
camada gold.

O catálogo de dados completo, com tipo, descrição, domínio, obrigatoriedade e linhagem de cada atributo, está na seção 4 do notebook.

5. Qualidade dos dados

A análise foi feita sobre a camada bronze, antes de qualquer tratamento.

Atributo	Dimensão	Verificação	Resultado	Tratamento
id	Completude e unicidade	Vazios e repetidos	0 vazios, 2.561 valores únicos	Nenhum necessário
id	Conformidade	Formato número_0 ou número_1	0 fora do padrão	Nenhum necessário
id × label	Consistência	Final _0 sempre legítima e _1 sempre golpe	Consistente em todas as linhas	Nenhum necessário
source	Conformidade	Categorias encontradas	sms (2.161) e fb (400)	Mantido
label	Conformidade	Só Legitimate e Smishing	2.009 e 552, sem outros valores	Traduzido (R2)
text	Completude	Textos vazios	0	Regra R4 mantida por segurança
text × label	Consistência	Mesmo texto com os dois rótulos	1 texto	Descartado (R5)
text	Unicidade	Texto e rótulo repetidos	0 antes da padronização	Repetições que surgem depois da R3 são removidas (R6)
text	Acurácia	Tamanho das mensagens	Legítimas de 5 a 477 caracteres; golpes de 44 a 185; 5 mensagens com menos de 10 caracteres	Mantidas, porque são plausíveis em SMS
text	Formatação	Quebras de linha e espaços repetidos	126 com quebra de linha e 436 com espaços repetidos	Padronizados (R3)
text	Dados pessoais	Sequências de 7 ou mais dígitos	1.007 mensagens	Anonimizadas (R8)

A base é um retrato fixo, publicado em 2025, e não recebe atualizações.

A base também é desbalanceada, com quase quatro mensagens legítimas para cada golpe. Isso não é um erro, porque reflete a prática. Mas explica por que a acurácia não é a métrica principal: um modelo que chamasse tudo de legítimo acertaria cerca de 78% das mensagens sem pegar nenhum golpe.

6. Transformação e carga
Regra	O que faz
R1	Renomeia as colunas para português: id_mensagem, fonte, texto e classe
R2	Traduz os rótulos: Smishing vira golpe e Legitimate vira legítima
R3	Troca quebras de linha e espaços repetidos por um espaço e remove espaços nas pontas
R4	Descarta textos vazios e classes fora do domínio
R5	Descarta textos que aparecem com as duas classes
R6	Mantém uma linha por texto, a de menor id
R7	Cria os atributos tamanho_caracteres, qtd_palavras, tem_numero, tem_link e menciona_nome, antes da anonimização
R8	Troca toda sequência de 7 ou mais dígitos por NUM_OCULTO
Conciliação
Item	Valor
Linhas na bronze	2.561
Linhas na silver	2.553 (2.006 legítimas e 547 golpes)
Linhas removidas pelas regras R4, R5 e R6	8
Textos da silver com número exposto	0
Faixa de caracteres	De 5 a 477
Faixa de palavras	De 1 a 83
7. Resultados

A silver foi dividida em 70% para treino (1.787 mensagens) e 30% para teste (766 mensagens: 164 golpes e 602 legítimas), com a mesma proporção de golpes nos dois conjuntos.

P1. Que características do texto diferenciam os golpes das mensagens legítimas?
Classe	Mensagens	Média de caracteres	Média de palavras	Com número	Com link	Menciona "nome"
golpe	547	95,5	16,3	98,2%	0,0%	89,0%
legítima	2.006	104,1	18,1	23,0%	1,5%	2,8%

Palavras mais frequentes, sem contar palavras comuns da língua:

Golpes: nome, valor, pesa, vem, manda, conta, numero, nr, ok, neste.
Legítimas: pesa, 00mt, valor, saldo, facil, conta, confirmado, novo, pm, 00.

Discussão. Os golpes da base seguem quase sempre o mesmo roteiro: "manda o valor neste número, vem em nome de fulano". O golpista finge ser um conhecido e pede que a transferência vá para outra conta. Por isso quase todos os golpes têm um número (98,2%) e a grande maioria menciona a palavra "nome" (89,0%).

As mensagens legítimas são, em boa parte, confirmações automáticas de transação ("confirmado", "saldo", "novo", valores em meticais). Como 23% delas também têm número, esse sinal sozinho não separa as classes.

Links quase não aparecem nos golpes desta base, ao contrário do golpe comum no Brasil, que costuma levar a um site falso. Para a empresa, o sinal mais útil é a combinação de pedido de transferência, um número e o nome de outra pessoa.

P2. Qual modelo deixa passar menos golpes, e quantos alarmes falsos cada um gera?

A árvore de decisão foi escolhida por ser o classificador visto em aula. Ela serve de base de comparação e é fácil de explicar.

O Naive Bayes foi escolhido por ser um modelo clássico de filtros de spam. Ele funciona bem com contagem de palavras e calcula a probabilidade de golpe, que a regra de decisão da P3 usa.

Conjunto de teste	Árvore de Decisão	Naive Bayes
Acurácia	0,97	0,96
Precisão (golpe)	0,939	0,843
Recall (golpe)	0,933	0,982
F1-score (golpe)	0,936	0,907
Golpes não detectados	11 de 164	3 de 164
Legítimas marcadas como golpe	10 de 602	30 de 602

Discussão. Pela acurácia e pelo F1-score, a árvore de decisão é um pouco melhor. Mas neste problema os erros não têm o mesmo peso. Deixar passar um golpe pode causar prejuízo, enquanto marcar uma mensagem legítima como golpe só atrasa a entrega. Por isso o critério de escolha é o recall da classe golpe.

O Naive Bayes deixou passar 3 golpes, contra 11 da árvore. Em troca, marcou 30 mensagens legítimas como golpe, contra 10 da árvore. Para segurança, essa troca vale a pena, e o Naive Bayes foi o modelo escolhido.

P3. Com a regra de decisão, quantos golpes são bloqueados e quanto trabalho sobra para a TI?

A regra de decisão usa o risco de golpe calculado pelo Naive Bayes:

risco de 70% ou mais: bloquear;
risco entre 40% e 70%: encaminhar para revisão da equipe de TI;
risco abaixo de 40%: liberar.

A árvore de decisão não serve para essa regra. Treinada sem limite de profundidade, ela quase sempre devolve 0% ou 100%, e sem valores intermediários não existe faixa de revisão.

Ação	Golpes	Legítimas	Total
Liberar	3	566	569
Revisar	0	9	9
Bloquear	161	27	188
Meta da hipótese	Meta	Resultado	Atendida
Golpes bloqueados	pelo menos 90%	98,2%	Sim
Legítimas bloqueadas	no máximo 5%	4,5%	Sim
Mensagens enviadas para revisão	no máximo 10%	1,2%	Sim

Discussão. As três metas foram atendidas, e a hipótese foi confirmada. A meta mais apertada foi a de mensagens legítimas bloqueadas: 4,5% para um limite de 5%.

O Naive Bayes dá riscos muito próximos de 0% ou de 100%, com poucas mensagens no meio. Por isso só 9 mensagens foram para revisão, o que reduz o trabalho da equipe de TI.

O lado ruim é que os erros acontecem com o modelo "confiante". As 27 mensagens legítimas bloqueadas tiveram risco alto e não caíram na faixa de dúvida, onde uma pessoa poderia corrigir o erro.

P4. Como a escolha do limite de bloqueio muda o resultado?
Limite de bloqueio	Golpes bloqueados	Golpes não bloqueados	Legítimas bloqueadas
0,50	161	3	30
0,60	161	3	29
0,70	161	3	27
0,80	161	3	22
0,90	161	3	20
0,95	161	3	15
0,99	159	5	13

Discussão. Entre 0,50 e 0,95, passam sempre os mesmos 3 golpes. O que muda é o número de mensagens legítimas bloqueadas, que cai de 30 para 15. Só a partir de 0,99 começam a passar mais golpes.

Para a empresa, isso indica que o limite de bloqueio poderia subir de 0,70 para 0,95. Isso reduziria quase à metade as mensagens legítimas bloqueadas, de 27 para 15, sem deixar passar nenhum golpe a mais no conjunto de teste. Como o teste tem só 164 golpes, a mudança deveria ser acompanhada pela equipe de TI antes de virar regra definitiva.

P5. O modelo reconhece golpes comuns no Brasil?

O teste usou seis golpes escritos no estilo dos que circulam no Brasil. A última mensagem imita o roteiro dos golpes da base, mas em português do Brasil.

Mensagem	Risco de golpe	Ação
Seu Pix foi bloqueado. Clique no link para regularizar sua conta	0,0%	Liberar
Correios: sua encomenda está retida. Pague a taxa de R$ 9,90 para liberar a entrega	0,1%	Liberar
Oi mãe, troquei de número. Pode fazer um Pix para mim? Depois te devolvo	0,0%	Liberar
Seu CPF está irregular. Regularize agora para evitar o bloqueio	26,2%	Liberar
Parabéns! Você ganhou um prêmio. Confirme seus dados bancários	0,1%	Liberar
Aquele valor manda neste número 61999990000, a conta está em nome de Maria Souza	100,0%	Bloquear

Discussão. Só 1 dos 6 golpes foi bloqueado, justamente o que imitava o roteiro dos golpes de Moçambique. O modelo só reconhece as palavras que viu no treinamento. Golpes com vocabulário brasileiro, como "Pix", "CPF" e "Correios", não se parecem com nada da base e receberam risco baixo.

A amostra é pequena e foi escrita à mão, então o resultado é um indicativo. Mesmo assim, a resposta é clara: o modelo não deve ser usado no Brasil sem ser treinado com mensagens daqui.

8. Discussão geral

A hipótese foi confirmada: o sistema bloqueou 98,2% dos golpes, bloqueou 4,5% das mensagens legítimas e mandou 1,2% das mensagens para revisão.

O ponto central do trabalho é a separação entre o modelo e a decisão:

o modelo calcula um risco;
a regra de decisão, com limites que a empresa pode ajustar, transforma esse risco em uma ação.

É isso que faz do projeto um sistema de suporte à decisão, e não só um classificador. Nos casos de dúvida, a decisão fica com uma pessoa.

As respostas se completam:

P1: os golpes da base seguem um roteiro bem marcado.
P2: por isso, um modelo simples como o Naive Bayes encontra quase todos eles.
P3 e P4: o sistema bloqueia a maior parte dos golpes com pouco trabalho de revisão, e o limite de bloqueio pode ser ajustado para reduzir os alarmes falsos.
P5: mostra o limite de tudo isso. O modelo só reconhece o tipo de golpe que viu, e um sistema de suporte à decisão depende da base de dados que o alimenta.
9. Ética, licença e proteção de dados

Licença dos dados. A base MOZ-Smishing é distribuída sob a licença CreativeML OpenRAIL-M, conforme a página do conjunto no Hugging Face. A licença permite uso e redistribuição, com restrições de uso descritas no próprio texto. O arquivo não é versionado neste repositório: o notebook baixa direto da fonte.

Dados pessoais (LGPD). As mensagens contêm números de telefone e nomes de pessoas. Por isso:

os números são anonimizados antes da carga na camada silver (R8);
nenhum texto original é exibido no notebook;
a camada bronze, que guarda o arquivo original, fica só na pasta privada do Colab ou do Google Drive.

Os nomes continuam no texto, porque removê-los com segurança exigiria uma técnica de reconhecimento de nomes, fora do escopo do MVP.

10. Autoavaliação

Perguntas respondidas. As perguntas P1 a P4 foram respondidas com os dados da base. A P5 foi respondida só em parte, porque o teste usou seis mensagens escritas por mim, e não uma amostra real de golpes brasileiros.

Limitações dos dados.

A base é de Moçambique, com vocabulário e tipos de golpe diferentes dos brasileiros.
A base é pequena, com cerca de 2,5 mil mensagens, e não recebe atualizações.
Os rótulos da fonte foram tomados como corretos.
Os nomes de pessoas não foram anonimizados.

O que eu faria diferente.

Executaria o pipeline em uma plataforma de dados na nuvem, como o Databricks, com as camadas gravadas como tabelas e a execução organizada em um job. Neste trabalho, o pipeline rodou no Google Colab e as camadas foram salvas como arquivos CSV.
Procuraria formar uma base brasileira, mesmo pequena, para treinar ou pelo menos testar o modelo.
Usaria validação cruzada em vez de uma única divisão entre treino e teste.

Extensões para uso contínuo.

Coletar, com consentimento, as mensagens que os funcionários reportam como suspeitas, e retreinar o modelo de tempos em tempos.
Agendar a execução do pipeline.
Criar um painel para a equipe de TI acompanhar as mensagens bloqueadas e revisadas.
Integrar a regra de decisão ao serviço de mensagens da empresa.
11. Referências

ALI, F. D. M. A.; SAIDE, S. M.; SOUSA-SILVA, R.; LOPES CARDOSO, H. MOZ-Smishing: a benchmark dataset for detecting mobile money frauds. In: WORKSHOP ON AFRICAN NATURAL LANGUAGE PROCESSING (AfricaNLP 2025), 6., 2025, Viena. Proceedings [...]. Viena: Association for Computational Linguistics, 2025.
