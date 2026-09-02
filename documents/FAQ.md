# FAQ do Laboratório

Este FAQ aborda os problemas de implantação e operação mais comuns ao executar o laboratório Agentic Applications for Unified Data Foundation com Microsoft Fabric e Azure AI Foundry.

## Antes da implantação

### Quais permissões o usuário de implantação precisa ter?

O usuário de implantação precisa de permissão para criar recursos do Azure, criar registros de aplicativo do Microsoft Entra e criar atribuições de função do Azure. `Owner` no escopo da assinatura é a configuração mais simples para o laboratório. Em um ambiente com menor privilégio, combine permissões de criação de recursos com `User Access Administrator` quando forem necessárias atribuições de função.

O Fabric também precisa de um membro do tenant que possa administrar a capacidade e o workspace. Um convidado B2B não pode ser administrador de capacidade do Fabric.

### Quais configurações de tenant do Fabric devem ser habilitadas?

No [portal de administração do Fabric](https://app.fabric.microsoft.com/admin-portal), habilite estas configurações de tenant para os usuários da implantação:

- **Ontology (preview)**
- **Graph (preview)**
- **Copilot and Azure OpenAI Service**

Aguarde até 15 minutos para a propagação das alterações antes de inicializar o cenário do Fabric.

### Quais ferramentas locais são necessárias?

Use PowerShell 7+, Azure Developer CLI (`azd`), Azure CLI (`az`), Bicep, Python 3.9+, Docker Desktop, Git e Microsoft ODBC Driver 17. Consulte o [Guia de Implantação](./DeploymentGuide.md) para conhecer os ambientes com suporte.

### Quanta capacidade do Fabric é necessária?

O laboratório requer uma capacidade Fabric F2 ou superior. A capacidade deve estar disponível em uma região compatível com as cargas de trabalho do Fabric usadas pelo cenário selecionado.

### Como escolho uma região?

Verifique todos os itens abaixo antes de provisionar:

1. Cota e disponibilidade regional do plano do App Service.
2. Disponibilidade do Azure AI Foundry e cota de modelo para as implantações de chat e de embeddings.
3. Disponibilidade da capacidade Fabric e das cargas de trabalho necessárias do Fabric.
4. Disponibilidade de Azure Container Registry, Azure AI Search, Storage, Cosmos DB e monitoramento.

A região padrão não tem necessariamente cota suficiente em todas as assinaturas. O guia [Quota Check](./QuotaCheck.md) explica a cota de modelos; as cotas do App Service também devem ser verificadas separadamente.

### Posso implantar recursos em mais de uma região?

Sim. Este laboratório dá suporte a uma implantação em regiões separadas quando uma única região não consegue hospedar todos os serviços necessários. Mantenha os recursos voltados ao aplicativo juntos sempre que possível e coloque Fabric, Foundry e Cosmos DB em uma região compatível que tenha capacidade suficiente. Configure os locais usando os parâmetros de ambiente `azd` documentados em [Customizing azd Parameters](./CustomizingAzdParameters.md).

### O Azure informa que a cota de VMs do App Service é zero. O que devo fazer?

Essa é uma restrição de cota da assinatura, não um erro do template. Verifique tanto a cota do SKU de App Service solicitado quanto a cota agregada **Total Regional VMs** da região. Se qualquer uma for zero, solicite cota pelo portal do Azure ou escolha outra região com capacidade. Aumentar o SKU do App Service não contorna uma cota regional agregada de VMs igual a zero.

### Minha implantação falha durante What-If ou preview. A implantação está quebrada?

Não necessariamente. Uma implantação aninhada grande pode exceder o limite de tamanho de solicitação do Azure Resource Manager para What-If, mesmo quando a implantação incremental normal funciona. Valide a compilação do Bicep e o mapeamento dos parâmetros e, se o erro de preview indicar especificamente limite de tamanho da solicitação, use o caminho normal de implantação incremental.

## Implantando e inicializando o cenário

### Por que `azd up` solicita valores que já estão no meu ambiente?

O `azd` pode solicitar valores de ambiente ausentes ou inválidos. Configure o ambiente primeiro com `azd env set` e execute o comando novamente. Evite colocar parâmetros Bicep com valor de array diretamente em um ambiente `azd`, a menos que o template aceite explicitamente uma cadeia JSON e faça seu parsing.

### A implantação da capacidade Fabric foi concluída, mas a inicialização do cenário falha. O que devo verificar?

Confirme que:

- A capacidade está ativa e possui no mínimo F2.
- O workspace está atribuído a essa capacidade.
- As configurações de tenant do Fabric estão habilitadas e já foram propagadas.
- A identidade que executa a inicialização é membro do tenant e tem acesso administrativo ao workspace e à capacidade.

### Por que minha conta de convidado não pode administrar a capacidade Fabric?

A administração de capacidade do Fabric exige uma conta membro do tenant. Use uma conta membro do tenant como administradora da capacidade e, depois, conceda ao convidado acesso ao workspace e aos recursos do Azure quando necessário.

### Por que o Azure AI Search retorna 403 durante a inicialização do cenário?

A identidade de inicialização precisa de permissões de plano de dados do Azure AI Search, não somente de plano de gerenciamento. Atribua **Search Index Data Contributor** para gravar documentos no índice; **Search Service Contributor** também pode ser necessário para ações de configuração no nível do serviço.

### Posso reutilizar um projeto Foundry ou workspace do Log Analytics existente?

Sim. Defina os parâmetros de ambiente correspondentes antes da implantação. Siga [Reusing an Existing Azure AI Foundry Project](./re-use-foundry-project.md) e [Reusing an Existing Log Analytics Workspace](./re-use-log-analytics.md) e confirme que a identidade de implantação tem as funções necessárias no recurso reutilizado.

## Login e chat web

### O aplicativo abre, mas enviar uma mensagem retorna 405. Qual é a causa?

O proxy reverso do frontend não está configurado para a API. Confirme que o App Service do frontend possui `BACKEND_API_HOST` configurado com o nome de host da API. O contêiner do frontend usa essa configuração para gerar o proxy `/api`; sem ela, o Nginx trata `POST /api/chat` como uma solicitação de site estático.

### O aplicativo abre, mas todas as mensagens falham após habilitar o login. O que devo verificar?

Use o endpoint `/api` da mesma origem no navegador. Não configure o navegador para chamar o App Service da API diretamente, pois isso ignora o proxy do frontend e o encaminhamento de token. Defina `APP_API_BASE_URL` como vazio/relativo e configure `BACKEND_API_HOST` no frontend.

### Por que OBO é necessário para perguntas ao Fabric Data Agent?

As consultas ao Fabric Data Agent são executadas com a identidade do usuário conectado. A API deve usar o fluxo On-Behalf-Of (OBO) do Microsoft Entra; identidade gerenciada não substitui esse caminho de acesso delegado ao Fabric. Siga [Set Up OBO Authentication](./SetupOBOAuthentication.md).

### O que o script de configuração OBO configura?

Ele cria ou configura um registro de aplicativo compartilhado, um escopo delegado `user_impersonation`, permissões de API e consentimento, EasyAuth nos dois App Services, armazenamento de token e configurações OBO na API. Alterações de autenticação podem levar até 10 minutos para serem propagadas.

### Um usuário conectado vê “An error occurred. Please try again later.” Como diagnostico?

Baixe os logs do App Service da API e relacione-os ao horário da solicitação. Verifique, nesta ordem:

1. O frontend encaminhou um token de acesso.
2. A API selecionou uma credencial OBO, em vez de usar identidade gerenciada como alternativa.
3. O usuário tem acesso ao workspace Fabric.
4. O usuário tem permissões de plano de dados no Azure AI Foundry.

Não dependa apenas da mensagem do navegador; o log da API identifica o serviço downstream e a ação negada.

### O log da API informa que falta `Microsoft.CognitiveServices/accounts/AIServices/agents/write`. Como corrigir?

Atribua **Foundry User** ao usuário no escopo do projeto Foundry. Se a atribuição no escopo do projeto não estiver disponível no seu ambiente, atribua-a no escopo da conta Foundry. Essa função de plano de dados do Foundry concede as ações de agente necessárias para criar conversas e execuções.

`Owner`, `Azure AI Developer` e `Cognitive Services OpenAI User` não concedem, sozinhos, as ações necessárias do Foundry Agent.

### Qual função Foundry um usuário do aplicativo deve receber?

Use **Foundry User** para pessoas que precisam criar ou testar no projeto e executar o fluxo do agente. Use **Foundry Agent Consumer** somente quando as permissões de consumo de endpoint forem suficientes. Use **Foundry Project Manager** ou **Foundry Owner** apenas para pessoas que precisam gerenciar o projeto ou ter responsabilidades administrativas mais amplas.

### Um usuário recebeu uma nova função, mas continua recebendo 403. E agora?

Aguarde a propagação do Azure RBAC e atualize a sessão do navegador ou saia e entre novamente. Verifique novamente o escopo da atribuição de função e o ID de objeto do principal antes de alterar o código do aplicativo.

## Gerenciamento de acesso

### De quais acessos um administrador do laboratório geralmente precisa?

Um administrador do laboratório normalmente precisa de acesso de gerenciamento de recursos do Azure, administração de capacidade e workspace Fabric, acesso ao projeto Foundry e funções relevantes de plano de dados para Search, Storage, Cosmos DB e ACR. Fora de um ambiente de laboratório, atribua apenas as funções necessárias para as responsabilidades da pessoa.

### A função Owner na assinatura concede acesso a todos os planos de dados?

Não. A função Owner no plano de gerenciamento do Azure não concede automaticamente acesso aos planos de dados de serviços como Azure AI Search, Storage, Cosmos DB, Fabric ou agentes Foundry. Atribua explicitamente as funções específicas de cada serviço.

### Quais funções geralmente são necessárias para inicializar os dados de exemplo?

Dependendo da operação, a identidade de inicialização pode precisar de:

| Serviço | Função típica |
|---|---|
| Azure AI Search | Search Index Data Contributor |
| Storage | Storage Blob Data Contributor |
| Cosmos DB | Cosmos DB Built-in Data Contributor |
| Azure Container Registry | AcrPush |
| Workspace Fabric | Admin ou Contributor |
| Projeto Foundry | Foundry User ou uma função Foundry mais ampla |

## Operação e recuperação

### Como confirmo que o aplicativo está saudável?

Verifique se os dois endpoints do App Service respondem com sucesso e teste:

1. Uma mensagem de chat simples, sem Fabric.
2. Uma pergunta de dados do Fabric, por exemplo, receita por ano.
3. Uma pergunta baseada em busca, se uma base de conhecimento estiver configurada.

Para solicitações autenticadas, confirme nos logs da API o encaminhamento de token do usuário, a seleção da credencial OBO e a conclusão bem-sucedida do streaming.

### O que devo coletar antes de relatar um problema?

Colete o horário aproximado em UTC, o usuário conectado, o prompt exato, a resposta do navegador e as entradas relevantes do log do App Service da API. Remova tokens de acesso, segredos de cliente, cadeias de conexão e outras credenciais antes de compartilhar logs.

### A configuração gerada do cenário deve ser versionada?

Não. A configuração gerada do cenário pode conter IDs de recursos específicos do ambiente e artefatos de implantação. Mantenha-a fora do controle de versão e documente apenas orientações de implantação reutilizáveis e sem informações sensíveis.

### Como excluo o laboratório?

Exclua o grupo de recursos somente quando não precisar mais da implantação e tiver confirmado que nenhum recurso compartilhado está sendo reutilizado. Consulte [Delete Resource Group](./DeleteResourceGroup.md).

## Documentação relacionada

- [Deployment Guide](./DeploymentGuide.md)
- [Fabric Deployment](./Fabric_deployment.md)
- [Set Up OBO Authentication](./SetupOBOAuthentication.md)
- [Azure Account Set Up](./AzureAccountSetUp.md)
- [Prerequisites and Costs](./PrerequisitesCosts.md)
- [Security Guidelines](./SecurityGuidelines.md)
