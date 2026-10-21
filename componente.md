Você vai me ajudar a criar, em .NET 9 / Blazor Server, um componente de FORMULÁRIO DINÂMICO orientado por metadados, mais um EDITOR VISUAL de leiaute para montar esses formulários. O caso de uso principal é um sistema de chamados em que cada tipo de chamado tem campos diferentes (ex.: tipo A tem CPF, endereço e telefone; tipo B tem CNPJ, nome e sobrenome), mas o componente deve ser genérico e servir para qualquer outro lugar.

Responda sempre em português do Brasil.

## Regra de trabalho (importante)

Antes de escrever qualquer código: leia o projeto, me faça as perguntas que faltarem, escreva o plano completo e AGUARDE meu pedido explícito para começar a codar. Não inicie a implementação por conta própria, mesmo que o plano pareça fechado. Depois que eu pedir, implemente por etapas (ordem no final) e pare ao fim de cada etapa para eu validar.

## Contexto e convenções (BBICore)

O componente deve morar na BBICore (Razor Class Library compartilhada). Antes de criar qualquer coisa, inspecione a BBICore e REAPROVEITE o que já existe: dropdown, combobox, botões, accordion, tabela/BzGrid, BzPopover, BBISeletorData (datas), BBIListaOrdenavel (reordenação por arrastar), BBiUpload (padrão de interfaces, ver abaixo), e o módulo Conversoes (extensões como texto.ParaInt()) e o namespace de máscaras (BBiCore.Mascaras). Só crie componente de entrada novo quando não houver equivalente.

Convenções a seguir:
- Componentes visuais `.razor` usam `@namespace BBiCore.Componentes`, com a lógica inline em `@code {}` (sem `.razor.cs`).
- Classes utilitárias não visuais usam namespaces específicos por pasta.
- JS fica ao lado, como `{Componente}.razor.js`, nunca em wwwroot.
- CSS: tokens do `bbicore.css` global + isolamento por componente em `.razor.css`. Não usar Bootstrap.
- Dependência externa de JS só se necessário (Floating UI é a única recomendada).

## Entidades por interface (mesmo padrão do BBiUpload)

O componente NÃO pode depender de classes concretas nem do Entity Framework. Antes de projetar, leia o BBiUpload na BBICore e REPRODUZA o mesmo padrão que ele usa para exigir uma interface básica das entidades de quem o consome (nomes, estilo, onde as interfaces ficam, como são restringidas nos parâmetros genéricos). Se o BBiUpload fizer diferente do que descrevo abaixo, siga o BBiUpload e me avise da divergência.

A BBICore define apenas as interfaces mínimas; cada projeto cria as próprias entidades e as faz implementá-las, podendo acrescentar quantas colunas quiser:

1. IDefinicaoCampo (o molde de um campo): id, nome interno (chave estável que nunca muda), rótulo, tipo do campo, obrigatório, ordem, seção, configuração em JSON (máscara, mínimo/máximo, tamanho, valor padrão, casas decimais), regra de visibilidade (método + parâmetros), validador (método + parâmetros), cálculo (método + parâmetros) e dados de leiaute (linha, coluna, largura em colunas).
2. IOpcaoCampo (opção de dropdown, radio ou grupo de checkbox): id, id do campo, código estável, texto, ordem, ativo, opção pai opcional (cascata, evolução futura).
3. IValorCampo (um valor preenchido): id de referência (id do chamado), id da definição do campo, ValorTexto, ValorNumero, ValorData, ValorBooleano e índice (para campos de vários valores).

O componente é genérico sobre essas interfaces, no estilo do BBiUpload: algo como BBiFormularioDinamico<TDefinicao, TValor, TOpcao>, com as restrições de interface nos parâmetros genéricos. Como o componente precisa criar valores novos para campos ainda não preenchidos, decida entre a restrição new() em TValor ou uma fábrica passada por parâmetro, e me explique a escolha no plano. Se o tipo do id (int, long, Guid) precisar ser genérico, proponha como tratar isso, também seguindo o que o BBiUpload faz.

As entidades concretas do exemplo (TipoChamado, CampoDefinicao, CampoOpcao, Chamado, ChamadoValor) e o mapeamento do EF Core ficam no projeto de exemplo/consumidor, NÃO na BBICore, implementando as interfaces acima:
- TipoChamado: id, nome, descrição, ativo, versão.
- Chamado: campos fixos (título, status, solicitante, datas), tipo, versão do tipo com que foi criado.
- ChamadoValor: implementa IValorCampo. Índices por chamado e por (campo, valor).

Regras de dados:
- Radio e dropdown gravam o CÓDIGO da opção (estável), nunca o texto.
- Grupo de checkbox grava uma linha por opção marcada (mesmo campo, índice diferente).
- Opções nunca são apagadas, só desativadas.
- Salvar é sempre em transação.
- Versionamento: alterar um tipo que já tem chamados cria nova versão; chamados antigos continuam abrindo com a versão original.

## Contrato do componente de formulário

O componente NÃO carrega nem salva nada. Quem usa carrega, passa e salva.

Parâmetros:
1. Definicoes: lista de TDefinicao do tipo de chamado (já inclui o leiaute).
2. Opcoes: lista de TOpcao dos campos que usam opções (ou as opções vêm de método provedor).
3. Valores: lista de TValor com binding bidirecional (@bind-Valores, com ValoresChanged). É a FONTE DA VERDADE do formulário.
4. ReferenciaId: id do chamado (vazio no cadastro novo).
5. Modo: Edicao, Visualizacao ou Cadastro novo.

Comportamento do binding da lista de valores:
- Ao carregar, cruza Definicoes e Valores campo a campo pelo id da definição. Achou valor: preenche. Não achou: vazio ou valor padrão (cobre chamado antigo aberto antes de um campo novo ser criado).
- Ao usuário alterar um campo, o componente atualiza a MESMA lista: altera o TValor existente, ou cria um novo (com ReferenciaId, id do campo e índice) se o campo ainda não tinha, e dispara ValoresChanged.
- Campo esvaziado, campo oculto por condição e opção de checkbox desmarcada saem da lista (remoção), para que quem salvar possa apagar a linha correspondente.
- Campos calculados também atualizam a lista, para o resultado ficar gravado e consultável por SQL.
- Exponha também um callback opcional com a diferença desde o carregamento: valores adicionados, alterados e removidos. Quem não quiser usar basta comparar a lista.
- No cadastro novo, o componente não define o id do chamado; o consumidor preenche ao salvar, na mesma transação que cria o chamado. Se ReferenciaId já estiver preenchido, os valores novos já nascem com ele.
- Saídas adicionais: callback de validade do formulário (válido ou não, com os erros) e método público para disparar a validação completa antes de salvar.
- Modo Visualizacao é somente leitura, com a formatação aplicada e um botão (ou parâmetro) para alternar para edição.
- Campo oculto por condição não é validado e seu valor é descartado ao salvar.
- O servidor continua revalidando e recalculando tudo ao salvar; o componente apenas oferece o método de validação, e o consumidor chama o mesmo caminho no servidor.

## Tipos de campo (mapa de tipos)

Mapa extensível "tipo do campo -> componente de entrada": texto, texto longo, número, moeda, data, data e hora, período, CPF, CNPJ, telefone, e-mail, dropdown, radio, grupo de checkbox, checkbox único, rótulo/texto informativo e campo calculado (somente leitura). Criar um tipo novo deve ser só registrar mais um item no mapa. Radio, dropdown e checkbox em grupo são "campos com opções": as opções vêm de IOpcaoCampo OU de um método provedor.

## Catálogo de métodos C#

As definições guardam apenas o NOME de um método e parâmetros em JSON. Há um catálogo com REGISTRO EXPLÍCITO (nunca reflexão livre sobre texto vindo do banco: só o que foi registrado pode ser chamado). A BBICore define a interface/contrato e o mecanismo de registro; cada projeto registra os seus métodos, além de um conjunto básico que já vem pronto.

Entrada padrão de todo método: um contexto com os valores atuais do formulário (por nome interno), a definição do campo, os parâmetros em JSON, o usuário logado e acesso aos serviços (IServiceProvider). Podem ser assíncronos.

Tipos de método: Validador (válido ou não + mensagem), Formatador (texto formatado), Calculador (valor do campo), Provedor de opções (lista para dropdown/radio/checkbox, ex.: vinda do banco) e Regra de visibilidade (verdadeiro/falso).

Métodos de fábrica: validar CPF, CNPJ, e-mail, telefone; formatar CPF, CNPJ, telefone, moeda; calcular soma, subtração, multiplicação, média e concatenação de campos informados pelos nomes internos; visibilidade por igualdade de valor de outro campo.

Comportamento:
- Validação ao sair do campo (não a cada tecla). Cálculo ao mudar um campo de que ele depende; métodos pesados com pequeno atraso.
- Campos se referenciam SEMPRE pelo nome interno.
- Campos calculados são somente leitura, recalculam por grafo de dependências e NÃO podem ter ciclos (detectar e impedir; a tela de configuração deve avisar).
- Falha controlada: método inexistente ou com exceção mostra erro claro no campo e não derruba a tela. Definições que apontam para método que sumiu do catálogo devem ser sinalizadas na tela de configuração.
- Validação entre campos (ex.: data final >= data inicial) é um validador de formulário com acesso a todos os valores.

## Leiaute visual configurável (parte central)

O leiaute é configurável por quem monta o tipo de chamado, sem desenvolvedor:
- O formulário é composto por SEÇÕES (com título, recolhíveis, usando o accordion da BBICore se couber), e cada seção por LINHAS.
- Cada linha tem de 1 a N campos lado a lado (ex.: linha 1 com um campo; linha 2 com três). Use uma grade de 12 colunas, onde cada campo tem uma largura em colunas (ex.: 12, 6+6, 4+4+4, 3+3+6).
- Em telas pequenas, as linhas empilham os campos automaticamente (responsivo).
- Cada campo pode ter, além da largura: rótulo acima ou ao lado, texto de ajuda, e se exibe ou não no modo Visualizacao.
- O leiaute fica gravado como dados (linha, coluna, largura, seção, ordem) na definição, e o renderizador apenas o interpreta.

## Editor visual do tipo de chamado (tela administrativa)

Crie uma tela/componente para o usuário final montar o tipo de chamado:
- Paleta de tipos de campo + tela de propriedades do campo selecionado (rótulo, nome interno, obrigatório, máscara, valor padrão, validador/formatador/calculador escolhidos de uma lista do catálogo com seus parâmetros, regra de visibilidade, opções).
- Montagem do leiaute adicionando seções, linhas e campos, arrastando para reordenar (reaproveite BBIListaOrdenavel) e ajustando a largura em colunas.
- Painel de PRÉ-VISUALIZAÇÃO ao vivo, mostrando o formulário exatamente como o usuário final verá, funcional (campos calculados, visibilidade e validações já respondendo), sem gravar nada. Deve poder alternar a pré-visualização entre largura de computador e de celular.
- Ao salvar, gera nova versão do tipo se já houver chamados abertos nele.
- Impedir configurações inválidas: nome interno duplicado, ciclo de cálculo, campo calculado referindo campo inexistente, opção sem código.

## Fora de escopo por enquanto

Expressões de fórmula em texto livre (só métodos do catálogo), opções em cascata (apenas deixar a opção pai prevista na interface), campos repetíveis (tabela interna), permissões por campo e histórico visual de versões.

## Ordem de construção (parar ao fim de cada etapa)

1. Interfaces da BBICore (seguindo o padrão do BBiUpload), entidades concretas e migração no projeto de exemplo.
2. Contrato do catálogo de métodos + registro + métodos de fábrica.
3. Componente de formulário com campos simples (texto, número, data) em cadastro novo, edição e visualização, já com leiaute em linhas e colunas e com o binding da lista de valores funcionando (adicionar, alterar, remover).
4. Campos com opções: dropdown, radio e grupo de checkbox (opções fixas e por provedor).
5. Validações e formatações pelo catálogo.
6. Visibilidade condicional e cálculos com grafo de dependências.
7. Editor visual do tipo de chamado com pré-visualização.
8. Página de exemplo no projeto de teste mostrando os tipos A e B do chamado, e revisão final dos pontos de segurança (catálogo explícito, revalidação no servidor).

Comece lendo a BBICore (em especial o BBiUpload), liste quais componentes existentes vai reaproveitar e quais faltam, faça suas perguntas e entregue o plano. Não codifique até eu pedir.
