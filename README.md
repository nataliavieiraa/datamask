# DataMask

Ferramenta interna de anonimização de dados sensíveis (LGPD) em bancos Oracle, que projetei e desenvolvi sozinha como DBA para resolver um problema real do meu dia a dia: dados de produção (CPF, nome, e-mail, endereço, dados financeiros) circulando sem controle em ambientes de desenvolvimento, teste e homologação.

> ⚠️ **Sobre este repositório**: a DataMask foi construída para uso interno da empresa onde trabalho — o código-fonte é propriedade da empresa e não está público. Este README descreve o problema, a solução e as decisões técnicas do projeto, como material de portfólio.

## O problema

Antes da ferramenta, mascarar dados sensíveis fora de produção era um processo manual, inconsistente e sem rastreabilidade: cada pessoa decidia na hora o que mascarar e como, sem garantia de que campos relacionados (como um CPF que aparece em várias tabelas) ficassem consistentes entre si, e sem nenhum histórico de quem fez o quê.

## O que a ferramenta faz

- **Descoberta automática** de colunas sensíveis: cruza o nome da coluna, o comentário cadastrado nela no Oracle (`COMMENT ON COLUMN`) e uma amostra dos dados reais pra sugerir a técnica de mascaramento certa, com um nível de confiança.
- **Mais de 25 técnicas de mascaramento**: geração de dado fictício plausível (nome, CPF, e-mail, endereço, cartão, placa...), hash determinístico, ruído numérico, embaralhamento entre linhas, generalização de data, redação de padrões sensíveis dentro de texto livre, entre outras.
- **Consistência garantida**: o mesmo valor original sempre vira o mesmo valor mascarado, em qualquer tabela ou execução — essencial pra não quebrar o relacionamento entre tabelas depois do mascaramento.
- **Consistência entre sistemas integrados**: suporte a importar o de-para que um sistema legado (fora do escopo da ferramenta) já gera para o próprio processo de anonimização dele, e reaproveitar exatamente os mesmos valores mascarados no Oracle — evitando que a mesma pessoa fique com dois valores fictícios diferentes em dois sistemas que trocam dado entre si.
- **Checagem de integridade referencial**: avisa quando uma coluna mascarada é referenciada por chave estrangeira em outra tabela com uma técnica diferente (ou nenhuma), com correção num clique.
- **Mascaramento condicional e incremental**: aplica uma técnica só quando outra coluna da mesma linha bate uma condição, e processa só as linhas novas desde a última execução (sem reprocessar tabelas inteiras a cada rodada).
- **Prévia antes de gravar**: mostra o valor original ao lado do valor que vai substituí-lo, linha por linha, antes de qualquer escrita real — com aviso automático quando um valor mascarado é maior que o tamanho da coluna de destino.
- **Backup automático opcional**: antes de gravar, salva uma cópia das linhas que serão alteradas numa tabela separada, como rede de segurança rápida.
- **Auditoria completa**: toda execução e toda mudança de configuração (quem, quando, o quê) fica registrada.
- **Controle de acesso por papel e por banco**: administrador x usuário comum, com escopo de acesso restrito a bancos específicos quando necessário.
- **Agendamento automático** de execuções recorrentes, sem depender de agendador externo.
- **Relatório de conformidade**: cobertura de mascaramento por banco/schema, distribuição por técnica, e alerta específico para regra cadastrada que nunca chegou a ser executada de verdade.

## Stack técnica

| Camada | Tecnologia |
|---|---|
| Linguagem | Python 3.12 |
| Framework web | Flask |
| Driver de banco | python-oracledb (modo thin, sem dependência de cliente Oracle instalado) |
| Dados fictícios | Faker |
| Agendamento | APScheduler |
| Frontend | HTML/CSS/JS servidos pelo próprio servidor (sem framework de frontend nem build step) |
| Armazenamento de configuração | CSV/JSON com escrita atômica e lock de concorrência (sem banco de dados próprio) |
| Testes | pytest — 358 casos automatizados, com o Oracle sempre simulado |

## Decisões técnicas que valem destacar

- **Sem banco de dados próprio, de propósito**: toda a configuração vive em CSV/JSON, com escrita atômica (arquivo temporário + substituição) e lock de concorrência — evita a complexidade operacional de manter um banco só pra guardar configuração de uma ferramenta interna, mantendo ainda assim segurança contra corrupção de dado em escrita concorrente.
- **Segurança de acesso ao Oracle**: todo identificador (schema/tabela/coluna) usado para montar uma consulta é validado por expressão regular antes de entrar no SQL — nunca é concatenado como texto livre vindo de input do usuário. Todo valor de dado trafega como bind variable.
- **Anonimização de mão única por padrão**: as técnicas de mascaramento não guardam nenhuma forma de reverter ao dado original — o cache que garante consistência (mesmo valor original sempre vira o mesmo valor mascarado) guarda as chaves como hash HMAC, nunca o dado real em texto puro.
- **Prevenção de colisão em colunas com restrição de unicidade**: quando uma técnica de substituição gera um valor que por acaso já está em uso, a ferramenta tenta gerar de novo automaticamente antes de desistir, evitando erro de violação de chave única no banco.
- **Desenvolvimento orientado a teste**: cada módulo de regra de negócio tem sua própria suíte de testes, sempre com o Oracle simulado — a suíte inteira roda sem precisar de um banco real disponível, o que permitiu evoluir e refatorar a ferramenta com confiança ao longo de várias rodadas de revisão de segurança e desempenho.

## Meu papel

Projetei, desenvolvi e mantenho essa ferramenta sozinha, como DBA — da concepção inicial até rodadas sucessivas de revisão de segurança, desempenho e usabilidade, incluindo testes end-to-end contra um ambiente Oracle real antes de cada entrega.

---

**Natália Vieira** — DBA
