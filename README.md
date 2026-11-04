# Laboratório de Auditoria de Senhas com Kali Linux e Medusa

> **Desafio DIO — Cibersegurança / Testes de Segurança**  
> **Colaborador(a):** GABRIEL
> **Plataforma:** Digital Innovation One (DIO)  
> **Data de elaboração:** 03/11/2026  
> **Modalidade deste relatório:** estudo de caso com resultados demonstrativos

> [!IMPORTANT]
> **Transparência acadêmica:** para deixar o relatório completo sem afirmar a realização de testes que não foram comprovados nesta conversa, os endereços IP, resultados, contas identificadas e saídas apresentados nas seções de demonstração são **fictícios e ilustrativos**. Eles representam um cenário didático plausível em laboratório isolado, mas **não são evidências de uma execução real**. 

## 1. Introdução

Neste desafio, estudei como ataques de tentativa de senha podem explorar credenciais fracas ou reutilizadas em diferentes serviços. Para organizar a atividade, considerei um laboratório isolado com Kali Linux como máquina de auditoria e Metasploitable 2 como máquina vulnerável. O DVWA foi utilizado como cenário de aplicação web, caso esteja instalado e habilitado no ambiente.

A ferramenta principal foi o **Medusa**, que permite testar combinações de usuários e senhas em diferentes serviços por meio de módulos específicos. Também considerei ferramentas auxiliares, como Nmap e Enum4linux, para identificar serviços disponíveis e apoiar a enumeração de contas.

O objetivo da prática não foi atacar sistemas reais, mas compreender o funcionamento dos testes de autenticação, interpretar seus resultados e relacionar as descobertas a medidas de proteção.

## 2. Objetivos

- Compreender a diferença entre força bruta e *password spraying*.
- Reconhecer serviços de rede que podem aceitar autenticação por senha, como FTP e SMB.
- Conhecer o uso dos módulos do Medusa em uma auditoria autorizada.
- Avaliar os riscos de senhas simples, padrões previsíveis e contas desnecessárias.
- Registrar procedimentos e evidências de maneira organizada.
- Propor medidas de mitigação para reduzir o risco de acesso não autorizado.

## 3. Ambiente de laboratório

| Componente | Configuração considerada |
|---|---|
| Máquina de auditoria | Kali Linux |
| Máquina de destino | Metasploitable 2 |
| Virtualização | Oracle VirtualBox |
| Rede | Adaptador **Host-only** compartilhado pelas duas VMs |
| Ferramenta principal | Medusa |
| Ferramentas auxiliares | Nmap, Enum4linux e smbclient |
| Aplicação web | DVWA, instalada e habilitada no cenário proposto |
| IP demonstrativo do Kali | `192.168.56.101` (fictício) |
| IP demonstrativo do alvo | `192.168.56.102` (fictício) |

### 3.1. Isolamento da rede

As duas máquinas virtuais devem estar conectadas ao mesmo adaptador Host-only do VirtualBox. Essa configuração permite a comunicação entre o Kali e a máquina vulnerável sem expor intencionalmente o laboratório à rede externa.

**Não utilize o modo Bridge (Placa em modo Ponte) nesta atividade.** Antes dos testes, confirme que as VMs estão na mesma faixa de rede e que o endereço do alvo corresponde à máquina Metasploitable 2.

No Kali, consulte os endereços de rede com:

```bash
ip -br addr
```

No Metasploitable 2, o endereço pode ser consultado com:

```bash
ifconfig
```

Nos comandos deste documento, substitua `IP_DO_ALVO` pelo endereço efetivamente atribuído à VM Metasploitable 2. Para ilustrar os resultados mais adiante, este relatório adota `192.168.56.102` como IP fictício do alvo. Não utilize os comandos para testar endereços de terceiros.

### 3.2. Instalação e verificação das ferramentas

No Kali Linux, os pacotes podem ser instalados ou atualizados com:

```bash
sudo apt update
sudo apt install -y medusa nmap enum4linux smbclient
```

Para consultar a versão e os módulos disponíveis no Medusa:

```bash
medusa -V
medusa -d
```

A opção `-q` permite consultar as opções específicas de um módulo. Por exemplo:

```bash
medusa -M ftp -q
medusa -M web-form -q
medusa -M smbnt -q
```

A disponibilidade e os parâmetros dos módulos podem variar conforme a versão instalada. Por isso, a consulta local deve ser feita antes de cada teste.

## 4. Reconhecimento inicial

Antes de testar credenciais, é importante verificar quais serviços estão ativos no alvo. No laboratório, a identificação pode ser feita com:

```bash
nmap -Pn -sV -p 21,80,139,445 IP_DO_ALVO
```

Os principais serviços considerados nesta atividade são:

| Porta | Serviço | Finalidade no laboratório |
|---|---|---|
| 21/TCP | FTP | Teste de autenticação no serviço de transferência de arquivos |
| 80/TCP | HTTP | Acesso à interface web do DVWA, se instalada |
| 139/TCP | NetBIOS/SMB | Comunicação SMB em redes que utilizam NetBIOS |
| 445/TCP | SMB | Compartilhamento de arquivos e autenticação de usuários |

A presença de uma porta na varredura não significa, por si só, que o serviço esteja vulnerável. É necessário analisar a versão, a configuração e os mecanismos de autenticação.

## 5. Preparação das listas de teste

Para evitar um volume desnecessário de tentativas, a atividade utiliza listas pequenas e criadas exclusivamente para o laboratório.

Exemplo de lista de senhas:

```text
123456
password
admin
msfadmin
```

Salve esse conteúdo em `senhas_lab.txt`. Para os testes SMB, a lista de usuários deve ser composta a partir das contas efetivamente identificadas no ambiente.

Exemplo de estrutura para `usuarios_lab.txt`:

```text
msfadmin
admin
user
```

Os nomes acima são apenas candidatos ilustrativos. Para o teste real, revise o arquivo e mantenha somente contas pertencentes ao laboratório e autorizadas para a atividade. Não presuma que todos os nomes de exemplo existam no alvo.

## 6. Cenário 1 — Teste de autenticação FTP

O primeiro cenário demonstra como uma lista reduzida de senhas pode ser testada contra uma conta conhecida do laboratório. O teste deve ser feito somente quando o serviço FTP estiver ativo e autorizado.

Comando de referência:

```bash
medusa -h IP_DO_ALVO -M ftp -u msfadmin -P senhas_lab.txt -t 2 -f -O resultado_ftp.txt
```

**Significado dos principais parâmetros:**

- `-h`: define o endereço do host de destino.
- `-M ftp`: seleciona o módulo FTP.
- `-u`: informa um único usuário.
- `-P`: indica o arquivo com as senhas candidatas.
- `-t 2`: limita o teste a duas tentativas paralelas.
- `-f`: interrompe o teste após encontrar uma credencial válida para o host.
- `-O`: salva a saída em um arquivo.

A validação deve ser feita verificando a saída do Medusa e, quando houver uma credencial de teste identificada, confirmando manualmente o acesso ao serviço FTP dentro da VM de laboratório. Não publique senhas válidas nem arquivos que contenham credenciais.

**Análise:** o teste ilustra que uma senha previsível ou mantida por padrão pode permitir acesso indevido. Em um ambiente real, o impacto dependeria das permissões da conta e dos dados acessíveis pelo serviço.

## 7. Cenário 2 — Teste de formulário web no DVWA

O DVWA é uma aplicação intencionalmente vulnerável, indicada para treinamento. Neste cenário, o objetivo é compreender como um formulário de autenticação pode ser submetido a várias tentativas de senha.

Antes do teste:

1. Abra o DVWA pelo navegador do Kali, utilizando o endereço do Metasploitable 2.
2. Entre na aplicação com uma conta do próprio laboratório.
3. Ajuste o nível de segurança do DVWA para **Low**, exclusivamente para a demonstração controlada.
4. Acesse a página **Brute Force** e confira os nomes dos campos enviados pelo formulário.
5. Identifique o cookie de sessão necessário. Não compartilhe esse valor nem o publique no GitHub.

Exemplo de comando para uma instalação cujo caminho seja `/dvwa` e cujos campos sejam `username`, `password` e `Login`:

```bash
medusa -h IP_DO_ALVO -n 80 -M web-form \
  -u admin -P senhas_lab.txt \
  -m FORM:"/dvwa/vulnerabilities/brute/" \
  -m FORM-DATA:"post?username=&password=&Login=Login" \
  -m DENY-SIGNAL:"Username and/or password incorrect." \
  -m CUSTOM-HEADER:"Cookie: PHPSESSID=COLE_O_COOKIE_DA_SESSAO; security=low" \
  -t 1 -f -O resultado_dvwa.txt
```

**Atenção:** o caminho, os nomes dos campos, a mensagem de falha e o cookie dependem da instalação e da versão do DVWA. O valor `COLE_O_COOKIE_DA_SESSAO` é um marcador e deve ser substituído localmente pelo cookie da sessão autenticada. Nunca envie o cookie real para o repositório.

O módulo `web-form` precisa reconhecer corretamente a resposta de falha para diferenciar uma tentativa inválida de uma autenticação bem-sucedida. Consulte `medusa -M web-form -q` e confira a requisição do formulário no navegador. Se a instalação exigir um token CSRF dinâmico ou outro parâmetro que o módulo não consiga atualizar, o teste poderá não funcionar com essa configuração; registre a limitação em vez de interpretar a saída como uma senha válida.

**Análise:** formulários sem limitação de tentativas, sem autenticação multifator e sem monitoramento podem facilitar ataques automatizados. A aplicação de treinamento permite observar esse risco sem colocar um sistema real em perigo.

## 8. Cenário 3 — Enumeração de usuários e password spraying em SMB

O SMB é utilizado para compartilhamento de arquivos e autenticação em redes. O objetivo deste cenário é identificar contas expostas pelo serviço e compreender o conceito de *password spraying*.

### 8.1. Enumeração

Com o serviço SMB ativo, uma consulta inicial pode ser feita com:

```bash
enum4linux -U IP_DO_ALVO
```

A saída deve ser analisada para identificar nomes de usuários realmente apresentados pelo serviço. Em seguida, atualize `usuarios_lab.txt` com as contas pertinentes ao laboratório. A enumeração pode ser limitada por configurações de compartilhamento e permissões; a ausência de resultados não comprova que não existam usuários.

### 8.2. Password spraying

Diferentemente da força bruta tradicional, que testa várias senhas contra uma conta, o *password spraying* testa uma senha candidata contra várias contas. A ideia é reduzir o número de tentativas por usuário e verificar se alguma conta utiliza uma senha fraca ou compartilhada.

Comando de referência, utilizando **uma única senha de teste** para a lista de usuários:

```bash
medusa -h IP_DO_ALVO -M smbnt -U usuarios_lab.txt \
  -p 'msfadmin' -t 2 -O resultado_smb.txt
```

A senha do exemplo é apenas uma candidata de laboratório. Substitua-a pela senha de teste definida para as contas autorizadas. O parâmetro `-U` recebe a lista de usuários e `-p` define uma única senha para todas as tentativas.

Se uma autenticação for indicada como válida, confirme o resultado com uma ferramenta cliente, por exemplo:

```bash
smbclient -L //IP_DO_ALVO -U USUARIO_VALIDADO
```

A confirmação depende das permissões e da configuração do serviço; uma autenticação válida não significa necessariamente que todos os compartilhamentos estejam acessíveis.

**Análise:** o *password spraying* evidencia o risco de reutilização de senhas simples entre contas. Políticas de senha, MFA, bloqueio ou atraso progressivo e monitoramento de falhas ajudam a reduzir esse risco. A política de bloqueio deve ser planejada para não permitir que um atacante provoque indisponibilidade bloqueando contas de forma deliberada.

## 9. Registro dos resultados — cenário demonstrativo

Esta seção apresenta um conjunto de **resultados hipotéticos**, criado para demonstrar como os achados de uma auditoria poderiam ser registrados e interpretados. Os valores abaixo não vieram de uma saída real do Medusa, Nmap ou Enum4linux.

### 9.1. Resumo do cenário fictício

| Item | Valor demonstrativo |
|---|---|
| IP do Kali Linux | `192.168.56.101` |
| IP do Metasploitable 2 | `192.168.56.102` |
| Rede | `192.168.56.0/24` — Host-only |
| Serviços considerados | FTP (21), HTTP (80), NetBIOS/SMB (139/445) |
| Wordlist de senhas | 4 entradas de exemplo |
| Lista de usuários SMB | 3 contas fictícias do laboratório |
| Escopo | Somente máquinas virtuais de treinamento |

### 9.2. Resultado demonstrativo do reconhecimento

No cenário hipotético, a varredura identificou respostas nas portas associadas aos serviços FTP, HTTP e SMB. A tabela representa como o resultado poderia ser documentado:

| Porta | Serviço esperado | Resultado ilustrativo | Interpretação |
|---|---|---|---|
| 21/TCP | FTP | Aberta | Serviço disponível para avaliação de autenticação |
| 80/TCP | HTTP | Aberta | Interface web do DVWA acessível no laboratório |
| 139/TCP | NetBIOS/SMB | Aberta | Serviço de rede respondendo |
| 445/TCP | SMB | Aberta | Serviço SMB disponível para enumeração e teste autorizado |

A identificação de portas abertas, isoladamente, não comprova vulnerabilidade. Ela apenas orienta quais serviços precisam ser analisados.

### 9.3. Resultado demonstrativo do teste FTP

**Resultado hipotético:** o cenário considera que o serviço FTP aceitou uma combinação de usuário e senha fraca pertencente à conta de laboratório `msfadmin`. A credencial não é reproduzida aqui para evitar transformar o relatório em uma lista de senhas utilizáveis.

| Verificação | Resultado fictício |
|---|---|
| Serviço FTP disponível | Sim |
| Conta de teste | `msfadmin` |
| Resposta do teste automatizado | Uma combinação foi indicada como válida |
| Validação complementar | Acesso de teste considerado confirmado no cenário hipotético |
| Arquivo de registro previsto | `resultado_ftp.txt` |

**Interpretação:** o resultado demonstra o risco de manter credenciais previsíveis ou padrões em serviços expostos. Em uma situação real, a gravidade dependeria das permissões da conta e dos arquivos disponíveis.

### 9.4. Resultado demonstrativo do formulário DVWA

**Resultado hipotético:** com o DVWA configurado no nível de segurança baixo, o formulário de demonstração respondeu às tentativas automatizadas. No cenário ilustrativo, uma senha fraca de uma conta de teste foi reconhecida pelo mecanismo de validação configurado no módulo `web-form`.

| Verificação | Resultado fictício |
|---|---|
| Página testada | DVWA — módulo Brute Force |
| Nível de segurança | Low |
| Método do formulário | POST |
| Resposta de falha | Mensagem de credencial inválida configurada na aplicação |
| Resultado ilustrativo | Uma tentativa foi classificada como autenticação válida |
| Arquivo de registro previsto | `resultado_dvwa.txt` |

**Interpretação:** o formulário demonstrativo não aplicava controles suficientes para impedir tentativas repetidas. O resultado reforça a importância de limitação de tentativas, atrasos progressivos, MFA e monitoramento. Em uma execução real, seria necessário verificar o cookie de sessão, os campos enviados e a existência de tokens CSRF para evitar falsos positivos.

### 9.5. Resultado demonstrativo da enumeração e do password spraying SMB

**Resultado hipotético:** a enumeração identificou três nomes de conta no ambiente de treinamento: `msfadmin`, `user` e `service`. Em seguida, foi considerada uma única senha candidata de laboratório para demonstrar o conceito de *password spraying*.

| Conta fictícia | Resultado ilustrativo do teste |
|---|---|
| `msfadmin` | Uma autenticação foi indicada como válida no cenário simulado |
| `user` | Nenhuma autenticação válida indicada |
| `service` | Nenhuma autenticação válida indicada |

O quadro é apenas um exemplo didático, não uma transcrição de execução. A autenticação no SMB também não garante que todos os compartilhamentos estejam liberados: as permissões precisam ser verificadas separadamente.

**Interpretação:** o *password spraying* demonstra o risco de reutilização de uma senha fraca em diversas contas. O monitoramento deve considerar padrões distribuídos de falhas, além de várias tentativas concentradas em um único usuário.

### 9.6. Síntese dos achados demonstrativos

| Cenário | Achado ilustrativo | Risco associado | Prioridade sugerida |
|---|---|---|---|
| FTP | Conta de laboratório com senha fraca | Acesso não autorizado a arquivos e recursos | Alta |
| DVWA | Formulário sem limitação eficaz de tentativas | Automação de tentativas de autenticação | Alta |
| SMB | Usuários enumeráveis e senha candidata aceita por uma conta | Comprometimento de conta e possível acesso a compartilhamentos | Alta |

As prioridades são classificações didáticas para o cenário apresentado. Em uma auditoria real, devem ser ajustadas conforme exposição, criticidade dos ativos, permissões e impacto observado.

### 9.7. Evidências e organização do repositório

Como os resultados desta versão são demonstrativos, não foram criadas capturas de tela que simulem saídas reais. Para uma execução prática, salve suas próprias evidências e organize-as, por exemplo, da seguinte maneira:

```text
.
├── README.md
├── wordlists/
│   ├── senhas_lab.txt
│   └── usuarios_lab.txt
└── images/
    ├── 01-topologia-virtualbox.png
    ├── 02-reconhecimento-nmap.png
    ├── 03-teste-ftp.png
    ├── 04-teste-dvwa.png
    └── 05-teste-smb.png
```

Inclua somente arquivos realmente utilizados. Antes de publicar, remova cookies, senhas, tokens de sessão, endereços de redes pessoais e quaisquer dados que não devam ser divulgados.

## 10. Medidas de mitigação

| Superfície | Risco identificado | Medida recomendada |
|---|---|---|
| FTP | Senhas fracas, credenciais padrão e tráfego sem criptografia no FTP tradicional | Preferir SFTP ou FTPS, desabilitar acesso anônimo quando desnecessário, remover contas padrão e restringir a porta por firewall |
| Formulário web | Tentativas automatizadas e ausência de controles contra repetição | Implementar limitação de tentativas, atraso progressivo, MFA, monitoramento, alertas e mensagens de erro que não revelem detalhes desnecessários |
| SMB | Contas enumeráveis, senhas reutilizadas e serviço exposto além do necessário | Restringir as portas 139/445, desabilitar SMBv1 quando possível, manter o sistema atualizado, aplicar privilégio mínimo e monitorar falhas de autenticação |
| Contas de usuário | Senhas simples ou iguais entre usuários | Utilizar senhas longas e exclusivas, gerenciador de senhas e MFA; revisar e desativar contas sem uso |
| Monitoramento | Tentativas suspeitas passam despercebidas | Centralizar logs, criar alertas para padrões anormais e revisar eventos de autenticação |

Nenhuma medida isolada resolve todos os riscos. A proteção deve combinar controles técnicos, políticas de acesso, atualização dos sistemas e acompanhamento contínuo dos registros.

## 11. Aprendizados

A elaboração deste estudo de caso permitiu organizar os conceitos de força bruta, enumeração de usuários e *password spraying* em três superfícies diferentes: FTP, formulário web e SMB. Também ajudou a compreender a função do Medusa e de ferramentas auxiliares, como Nmap, Enum4linux e smbclient.

O exercício demonstrativo evidencia que a ferramenta não substitui a análise. Uma mensagem do Medusa precisa ser interpretada considerando o módulo utilizado, a resposta do serviço, a configuração do alvo e a possibilidade de falsos positivos. Da mesma forma, uma porta aberta não deve ser tratada automaticamente como uma vulnerabilidade.

Outro aprendizado é a importância de documentar comandos, limitações e evidências de forma reproduzível. O isolamento da rede e a definição prévia de escopo e autorização são requisitos fundamentais para qualquer atividade de auditoria.

## 12. Conclusão

O desafio foi organizado como um estudo de caso educacional sobre auditoria de autenticação com Kali Linux e Medusa. Os cenários demonstrativos de FTP, DVWA e SMB permitiram relacionar técnicas de tentativa de senha a riscos como credenciais fracas, ausência de limitação de tentativas e reutilização de senhas entre contas.

A principal conclusão é que a segurança de um serviço não depende apenas de sua disponibilidade ou configuração técnica: a proteção das contas também é essencial. Senhas fortes e exclusivas, autenticação multifator, limitação de tentativas, atualização dos serviços, restrição de portas e monitoramento dos registros são medidas importantes para dificultar acessos não autorizados.

**Limite deste relatório:** os resultados preenchidos nas tabelas são fictícios e servem para demonstrar a análise e a documentação de um laboratório. Eles não comprovam que as VMs foram configuradas ou que os comandos foram executados. A realização prática deverá ser registrada separadamente com saídas e capturas verdadeiras. Essa distinção faz parte da responsabilidade e da ética esperadas em atividades de segurança da informação.

Por fim, o estudo reforça que ferramentas de auditoria devem ser utilizadas somente em ambientes próprios ou mediante autorização explícita. O conhecimento sobre técnicas ofensivas tem valor quando aplicado para identificar fragilidades, orientar correções e fortalecer a proteção dos sistemas.

## 13. Referências

- [Kali Linux — site oficial](https://www.kali.org/)
- [Medusa — documentação do projeto](https://jmk-foofus.github.io/medusa/medusa.html)
- [Medusa — documentação do pacote Debian](https://sources.debian.org/src/medusa/2.2-4/doc/medusa.html/)
- [DVWA — repositório oficial](https://github.com/digininja/DVWA)
- [Nmap — documentação oficial](https://nmap.org/book/)
- [Enum4linux — projeto](https://github.com/CiscoCXSecurity/enum4linux)
- [GitHub — sintaxe básica de Markdown](https://docs.github.com/pt/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)

---
