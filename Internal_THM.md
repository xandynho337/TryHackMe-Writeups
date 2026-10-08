# Writeup: Internal - TryHackMe

**Dificuldade:** Difícil  
**OS:** Windows  
**Categorias:** Active Directory, Privilege Escalation e Web   
**Data de Conclusão:** 20/10/2025  

---
## Resumo Executivo
> Visão geral do impacto técnico e de negócio das vulnerabilidades encontradas.

O objetivo deste laboratório foi obter acesso com privilégios máximos (NT Authority\SYSTEM`) no alvo. A intrusão inicial foi viabilizada pelo vazamento de credenciais de domínio em um repositório público do GitHub (Information Disclosure). De posse do acesso inicial, foi realizada a enumeração do Active Directory e a execução da técnica de Kerberoasting, permitindo a extração e quebra offline da credencial de uma conta de serviço com acesso RDP. Por fim, a escalada de privilégios locais para NT AUTHORITY\SYSTEM ocorreu pela exploração do vetor Unquoted Service Path no serviço ZeroTier One.

---

## 1. Reconhecimento & Enumeração

### Varredura de Portas (Nmap)
Sintaxe utilizada para a varredura inicial:

```bash
nmap -v -sSCV 10.66.153.84 -Pn -g 80 -D RND:30 --top-ports=5000
```

<img width="732" height="254" alt="image" src="https://github.com/user-attachments/assets/9220c738-8fe5-4776-a48a-f1912da5f92b" />

**Portas Abertas Encontradas relevantes:**
* **22/tcp:** ssh - OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
* **80/tcp:** http - Apache/2.4.29

Após a varredura é necessário realizar a enumeração dos serviços e tentativa de interação com os mesmos.

### Enumeração do Serviço Web
Ao acessar o site, é recebido a página padrão do Apache, assim, será necessário realizar a enumeração de diretórios e arquivos do servidor web, na tentativa de encontrar um ponto de entrada explorável.

<img width="1364" height="626" alt="image" src="https://github.com/user-attachments/assets/54d15786-0d73-41b6-80e5-37b4609bb7e7" />

A enumeração de diretórios e arquivos foi realizada com a ferramenta Feroxbuster e uma wordlist do dirb.

```bash
feroxbuster -u "http://10.66.153.84/" -w /usr/share/wordlists/dirb/big.txt -a "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/149.0.0.0 Safari/537.36" -C 503
```

<img width="886" height="504" alt="image" src="https://github.com/user-attachments/assets/4ea39ec2-ed12-40e7-abcb-ce41c0850498" />

A ferramenta trouxe diversas pastas e subpastas do servidor web, realiza-se a divisão em diretórios pais, e seus filhos, para explorar um de cada vez.
 
**Diretórios Encontrados:**
* **/blog:**
* **/javascript:** 
* **/phpmyadmin:**
* **/wordpress:**  

### Enumeração do diretório "/blog":

Ao acessar o diretório, é possível notar que a página vem quebrada, isso pode ocorrer quando referenciais dentro do HTML não foram tratadas.

<img width="1060" height="625" alt="image" src="https://github.com/user-attachments/assets/ec207377-276c-4456-a0cc-b728c66e0cfb" />

Assim, será necessário avaliar o código HTML dessa página em busca de nomes de domínios para inserção nos hosts da máquina do atacante.

<img width="1112" height="563" alt="image" src="https://github.com/user-attachments/assets/954ec902-80d4-4703-baeb-7bd0394386bd" />

Verificando o código HTML do site, foi encontrado um nome de domínio para a máquina referente ao Try Hack Me(THM). O mesmo foi inserido no arquivo hosts do atacante e a página recarregada para análise.

<img width="443" height="139" alt="image" src="https://github.com/user-attachments/assets/394ea097-b0b4-49d0-aa72-bee3fd454f79" />
<img width="644" height="608" alt="image" src="https://github.com/user-attachments/assets/ebe11f65-3df5-49ad-b770-ba1ff0c8b91c" />

As quebras do site foram solucionadas após a inserção do domínio no arquivo hosts. Agora pode-se analisar de forma correta o diretório blog e sua HTML em busca de pontos de entrada.

<img width="1072" height="481" alt="image" src="https://github.com/user-attachments/assets/5227aae5-c528-4a08-bfb8-e1da32090ab7" />

---
## 2. Intrusão Inicial (Initial Access)

Foi encontrado um número de versão do wordpress. Sabendo disso, será utilizado uma ferramenta para a enumeração de pontos de entrada e vulnerabilidades no wordpress, o wpscan, buscando por plugins e contas de usuários vulneráveis:

```bash
wpscan --url http://internal.thm/blog/ -e vp,u --plugins-detection aggressive --api-token [TOKEN]
```
Não foram encontrados plugins interessantes que poderiam permitir um acesso ao servidor, mas, foi encontrado um nome de usuário vulnerável.

<img width="886" height="443" alt="image" src="https://github.com/user-attachments/assets/a3e340b4-bdd5-4c23-b031-0acf1d0e9599" />

Com esse usuário, é possível utilizar o modo de Brute Force dessa ferramenta e verificar se achamos uma credencial válida para o mesmo.

```bash
wpscan --url http://internal.thm/blog/ --usernames admin --passwords /usr/share/wordlists/rockyou.txt --max-threads 50 --api-token [TOKEN]
```

<img width="889" height="236" alt="image" src="https://github.com/user-attachments/assets/33765503-74e8-4dbd-922c-51201ab3243c" />

Foi encontrado uma credencial válida de acesso administrativo para acesso ao WordPress do servidor. Com as credenciais em mãos, foi feito login no painel administrativo.

<img width="1357" height="581" alt="image" src="https://github.com/user-attachments/assets/7a82163b-fe6b-4fa6-a659-5269c0df1b55" />

Com esse acesso, analisa-se os plugins e temas em busca de algum que permita escrita, dessa forma, pode-se realizar um Reverse Shell para a máquina do atacante.

### Exploração da Vulnerabilidade
* **Vulnerabilidade:** Remote Code Execution via Edição Arbitrária de arquivos de Temas
* **Vetor de Ataque:** Acesso administrativo e permissão de edição de arquivos de tema do wordpress levando a um RCE.

Após a entrada no painel administrativo do WordPress, foi encontrado um tema que permitia sobrescrever o arquivo php do mesmo. 

<img width="1356" height="578" alt="image" src="https://github.com/user-attachments/assets/8d65f85e-bec5-42b2-8254-1d764f5521c8" />

Assim, é utilizado essa vulnerabilidade para usar um shell reverso no lugar do corpo original do texto e salvar o arquivo .php com a nova informação(O Reverse Shell utilizado, foi pego no site revshells.com).

<img width="1314" height="549" alt="image" src="https://github.com/user-attachments/assets/0d266c9b-e9ee-478d-b9de-e4de70e16771" />

Agora com o payload já pronto para ser executado, é preciso abrir uma mesma porta inserida nele na máquina do atacante.

```bash
nc -vlp 1234
```

<img width="381" height="137" alt="image" src="https://github.com/user-attachments/assets/9ae80701-b975-4ac7-b812-0b55932ff4f2" />

Com a porta já aberta aguardando a conexão na máquina do atacante, para que a shell interativa seja executada, é necessário acessar o arquivo "archive.php" do tema, diretamente pelo navegador, que fará o payload ser executado e estabelecer conexão com a máquina do atacante. O caminho padrão do tema vulnerável é "/wp-content/themes/twentyseventeen/"

<img width="1241" height="446" alt="image" src="https://github.com/user-attachments/assets/0c50d900-47a9-411e-ba30-cf02cc903421" />

Acesso concedido ao servidor com usuário www-data!!

### Exploração do Servidor manualmente

Após o acesso, em primeiro lugar foi realizado uma tentativa de adentrar ao usuário que existia na máquina, mas o acesso não tinha permissão, com isso em vista, realizamos o processo de melhoria da shell que foi obtida.

```bash
python3 -c ‘import pty; pty.spawn(”/bin/bash”)’
#Após o Enter, fazer o comando CTRL+Z
stty raw -echo; fg #Aqui precisa apertar Enter duas vezes
stty rows 16 columns 136
export TERM=xterm-256color
```

<img width="526" height="252" alt="image" src="https://github.com/user-attachments/assets/c29521b1-96a6-4aa0-9ca9-8a0b0f63b05e" />

Agora com uma shell completa, será necessário procurar novos pontos de entrada ou credenciais que permita o acesso ao usuário "aubreanna".

**Diretórios Interessantes para Exploração:**
* **/tmp:**
* **/opt:** 
* **/var/www/html/:**

Ao analisar o conteúdo das pastas, uma chamou mais atenção, essa pasta tinha um arquivo "containerd" e um "wp-save.txt". Ao verificar esse arquivo, conseguimos as credenciais do usuário aubreanna.

<img width="885" height="421" alt="image" src="https://github.com/user-attachments/assets/ef10300a-ec25-45ae-9ec8-91fe27d48180" />

Ao testar as credenciais na porta SSH do servidor, foi constatado que era possível o acesso. Dessa forma, conseguimos a primeira flag que era buscado "THM{[ENCONTRADO]}"

```bash
ssh aubreanna@internal.thm
```

<img width="713" height="527" alt="image" src="https://github.com/user-attachments/assets/669764dd-3dfd-41e1-8b0e-b5400e24dee2" />

---

> **Acesso obtido:** Acesso inicial realizado via Brute Force no usuário admin do Wordpress, após RCE via edição de temas do mesmo, levando ao SSH por Information Disclosure dentro do servidor.

---

## 3. Escalada de Privilégios (Privilege Escalation)

### Enumeração Interna
Identificação do vetor de elevação de privilégios:

No mesmo usuário, há um arquivo "jenkins.txt" que informa existir um serviço Jenkins Interno rodando em "172.17.0.2:8080". Mas, analisando a máquina que foi acessada, ela possui o IP 172.16.0.1 atribuída.

<img width="885" height="448" alt="image" src="https://github.com/user-attachments/assets/fdb2b0db-3492-4a42-aede-1acfa8e6130b" />

Dessa forma, para conseguir interagir com o serviço interno, será necessário realizar um tunelamento SSH na máquina, usando as credenciais do usuário "aubreanna".

```bash
ssh -L 1234:172.17.0.2:8080 aubreanna@internal.thm
```

Após realizar o tunelamento, será possível acessar o serviço pela máquina do atacante na porta escolhida no ip local da máquina.

<img width="931" height="607" alt="image" src="https://github.com/user-attachments/assets/1c4763a1-44a6-4d9f-b5fa-1d1454994968" />

Agora com acesso ao serviço, será necessário descobrir credenciais para acesso ao mesmo. Por se tratar de um serviço interno, será realizado um ataque de Brute Force com FFUF. Para isso, utiliza-se o navegador do Burp para capturar a requisição de login incorreta.

<img width="1287" height="615" alt="image" src="https://github.com/user-attachments/assets/f7ed3c51-1d13-4529-a374-6dcee2e6786b" />

Com isso, é necessário realizar a configuração no ffuf, para que o mesmo teste as senhas até que a resposta seja diferente de "302-401".

```bash
ffuf -X POST -u http://127.0.0.1:1234/j_acegi_security_check -d "j_username=admin&j_password=FUZZ&from=%2F&Submit=Sign+in" -H "Content-Type: application/x-www-form-urlencoded" -w /usr/share/wordlists/rockyou.txt -r -fc 401
# -r faz com que o quando o ffuf encontre um 302, ele continue.
```

<img width="827" height="439" alt="image" src="https://github.com/user-attachments/assets/396c7d8a-fdf1-4ad9-97a9-14dcdb24fbc1" />

Assim foi possível acessar o painel de controle do Jenkins. Ao entrar e verificar as configurações de gerenciamento do Jenkins, é possível ver uma opção de "console de script", assim, podendo ser utilizado para executar uma shell reversa, dando acesso não autorizado ao host que executa o serviço.

<img width="1076" height="588" alt="image" src="https://github.com/user-attachments/assets/4d14a043-b8d0-4155-b98d-5411a0a39007" />

Com isso, ao acessar o site do revshells.com, foi possível encontrar um script em Groovy, que será utilizado para a tentativa de shell reverso com o jenkins.

<img width="1108" height="651" alt="image" src="https://github.com/user-attachments/assets/553c5148-9223-497b-819c-d3a53ad4eb03" />

Ao executar o script com o Jenkins, foi possível acessar a máquina que hospeda o Jenkins, e dessa forma, foi realizado a tentativa de chegar ao diretório root. Não foi possível, então será necessário realizar a mesma enumeração de diretórios comuns anteriormente feito no usuário aubreanna.

<img width="1038" height="625" alt="image" src="https://github.com/user-attachments/assets/053c9423-6921-4371-be99-3aa8c7593168" />

Ao realizar a enumeração, foi possível verificar um arquivo com o nome de "note.txt" dentro da pasta "/opt". Dentro desse arquivo havia as credenciais de root da máquina. Com essa credencial, foi feito a tentativa de conexão com o usuário root no SSH.

<img width="412" height="131" alt="image" src="https://github.com/user-attachments/assets/2cf76839-7c81-4ec9-805c-fdbbdb8deb74" />

Escalação de privilégios bem sucedida, foi possível acessar o SSH da máquina com o usuário root!

---

> **Acesso obtido:** Acesso Shell via SSH com o usuário root.

---

## 4. Recomendações de Mitigação (Remediação)

1. **SMB(Compartilhamento de Arquivos):** Desabilitar o acesso anônimo (Guest / Anonymous Access) aos compartilhamentos de rede SMB e restringir a visibilidade de pastas com dados sensíveis aplicando permissões adequadas de ACL/DACL.
2. **Vazamento no GitHub:** Implementar ferramentas de Secret Scanning (ex: GitGuardian, Trufflehog) na esteira de CI/CD, revogar imediatamente a credencial comprometida e sanitizar o histórico de commits no repositório.
3. **Contas de Serviço / SPNs (Kerberoasting):** Adotar senhas fortes/complexas (com mais de 25 caracteres) para contas de serviço ligadas a SPNs para inviabilizar ataques de força bruta, ou migrar para Group Managed Service Accounts (gMSA).
4. **Serviços no Windows (PrivEsc):** Envolver todos os caminhos de executáveis de serviços que contenham espaços entre aspas no Registro do Windows (HKLM\SYSTEM\CurrentControlSet\Services) e aplicar o princípio do menor privilégio nas permissões de diretórios no sistema de arquivos (NTFS).
