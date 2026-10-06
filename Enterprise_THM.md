# Writeup: Enterprise - TryHackMe

**Dificuldade:** Difícil  
**OS:** Windows  
**Categorias:** Active Directory e PrivEsc  
**Data de Conclusão:** 06/10/2025  

---

## Resumo Executivo
> Visão geral do impacto técnico e de negócio das vulnerabilidades encontradas.

O objetivo deste laboratório foi obter acesso com privilégios máximos (`root` / `SYSTEM`) no alvo. A intrusão inicial ocorreu através de [Nome da Vulnerabilidade / Vetor], permitindo [impacto, ex: Execução Remota de Código (RCE)]. A escalada de privilégios foi alcançada explorando [Mecânica/Falha de Configuração].

---

## 1. Reconhecimento & Enumeração

### Varredura de Portas (Nmap)
Sintaxe utilizada para a varredura inicial:

```bash
nmap -v -sSV 10.66.166.228 -T2 -g 80 -D RND:30 --top-ports=500 --open
```

<img width="832" height="341" alt="image" src="https://github.com/user-attachments/assets/3bc30fc7-3d61-4913-a52d-2923c7d941ae" />

Após a varredura inicial comum, pegamos as portas abertas mais interessantes agora conhecidas e refazemos a varredura utilizando os scripts do próprio nmap utilizando a flag "-sSC".

```bash
nmap -v -sSC 10.66.166.228 -T2 -g 80 -D RND:30 -p50-470,3268,3389,5985 --open
```

<img width="824" height="459" alt="image" src="https://github.com/user-attachments/assets/d01d25c8-263a-4448-91e0-4145d32a7b13" />

Assim foi possível encontrar nomes de domínios relevantes que serão adicionados no arquivo "/etc/hosts" da máquina do atacante para próximos passos.
<img width="824" height="473" alt="image" src="https://github.com/user-attachments/assets/856512c2-724f-4009-9ac0-3deb0a9b88ca" />

**Portas Abertas Encontradas relevantes:**
* **80:** http
* **135:** msrpc
* **139:** netbios-ssn
* **445:** microsoft-ds(smb)
* **3389:** ms-wbt-server
* **5985:** wsman

Após a varredura é necessário realizar a enumeração dos serviços e tentativa de interação com os mesmos.

### Enumeração do Serviço Web
Ao acessar o site, apenas um página em branco com uma escrita informando para não ser mexido é mostrado em todas os domínios que encontramos anteriormente.

<img width="1365" height="626" alt="image" src="https://github.com/user-attachments/assets/9195332b-a34d-4273-9f70-97b32fb47636" />

Dessa forma, foi feito a tentativa de encontrar diretórios e arquivos nesse site com o feroxbuster, mas o mesmo retornou apenas o arquivo "robots.txt", mas ao acessa-lo, não havia informações que podíamos usar.

```bash
feroxbuster -u "http://enterprise.thm/" -w /usr/share/wordlists/dirb/big.txt -a "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/149.0.0.0 Safari/537.36" -C 503
```

<img width="831" height="425" alt="image" src="https://github.com/user-attachments/assets/7a18cb88-2677-46ac-9aff-bd3cd95c6338" />

<img width="1359" height="626" alt="image" src="https://github.com/user-attachments/assets/fb38f86a-6740-4292-bb10-3f6bb46bb9fd" />

Dessa forma, partimos para enumeração de outro serviço.

### Enumeração do Serviço SMB
Para realizar a enumeração no SMB, foi necessário realizar a tentativas de uso de usuário anônimo sem senha. Ao realizar a tentativa encontramos o primeiro ponto de entrada.

```bash
smbclient -L \\10.67.160.138 -U
```

<img width="832" height="386" alt="image" src="https://github.com/user-attachments/assets/76b9fe31-36bd-41e2-8f55-2205ab5afe5e" />

Ao realizar a listagens dos diretórios disponíveis no SMB, encontramos dois relevantes: "Docs" e "Users" que possui uma observação para que não seja tocado, então, será utilizado o smbclient para explorar as pastas.

```bash
smbclient //enterprise.thm/Users
# Após entrar no SMB, exportar tudo para sua máquina
smb: \> ls #Para listar
smb: \> cd #Para entrar
smb: \> get #Para baixar algum arquivo
```

<img width="826" height="416" alt="image" src="https://github.com/user-attachments/assets/fa397b6b-0f3a-4f85-b41f-7d7d623928e1" />

Verificando todas as pastas, as duas únicas pastas que permite a exploração/listagem são as pastas "Default" e "LAB-ADMIN". Sabendo disso, guardamos a informação caso seja necessário usa-la novamente depois.


## 2. Intrusão Inicial (Initial Access)


Após não encontrar nada relevante para a intrusão nos diretórios encontrados no SMB, voltamos à etapa de varredura, e vamos realizar uma nova varredura em busca de mais portas abertas, e dessa vez, realizamos para todas as portas, afim de encontrar uma porta da qual não encontramos antes.
<img width="834" height="468" alt="image" src="https://github.com/user-attachments/assets/181da457-7496-4e07-ad27-a2a332a4f659" />

Essa nova varredura descobriu diversas novas portas mas que possivelmente estão sendo filtradas, mas, uma delas dispõe de um serviço IIS, dessa forma, foi analisado na web essa porta.
<img width="1365" height="588" alt="image" src="https://github.com/user-attachments/assets/4513e690-a40d-44f2-a4b3-6a8b75691151" />

Ao acessar a interface encontramos uma página que espera uma conexão com e-mail, que também dispõe de conexões com Google e Microsoft. Mas antes mesmo de tentar acessar com uma conta teste, há um aviso na tela informando a migração para o GitHub, algo estranho.
Com isso em mente, foi pesquisado usando Google Dorks relação com o GitHub e o enterprise.thm. Logo no primeiro resultado foi encontrado um repositório relacionado à máquina do laboratório.
<img width="1364" height="582" alt="image" src="https://github.com/user-attachments/assets/aeb487e5-7516-4dc8-8baf-e97269383a7b" />

Explorando o GitHub, foi possível encontrar 1 pessoa ligada ao repositório do Enterprise. analisando essa pessoa, encontramos um arquivo .ps1. Ao verificar esse arquivo não foi encontrado nada, mas ao verificar o histórico de commits, foi descoberto credenciais de AD.
<img width="1360" height="583" alt="image" src="https://github.com/user-attachments/assets/17b5b9d5-3327-4e71-85a9-012ee29a3da8" />

### Exploração da Vulnerabilidade
* **Vulnerabilidade:** Credenciais de domínio expostas em repositórios públicos
* **Vetor de Ataque:** Uso das credenciais para busca de novas credenciais no AD, enumeração de usuários e acesso não autorizado via RDP.

Agora por termos credenciais válidas de um usuário, podemos usá-la para encontrar outros usuários e senhas procurando por SPNs no domínio e tentar a quebra do mesmo com john, afim de conseguir mais usuários.
```bash
/usr/share/doc/python3-impacket/examples/GetUserSPNs.py lab.enterprise.thm/n[ENCONTRADO]:T[ENCONTRADO] -request
```
<img width="834" height="268" alt="image" src="https://github.com/user-attachments/assets/f0f58d92-1071-4b43-bc10-5f45b94b66de" />

Após conseguir a hash SPN, deve ser utilizado o john para a tentativa de quebra conseguindo uma nova credencial nesse domínio.
```bash
john --format=krb5tgs spn --wordlist=/usr/share/wordlists/rockyou.txt
```
<img width="796" height="180" alt="image" src="https://github.com/user-attachments/assets/58955f1b-b876-4565-bfe3-e819db62675f" />

Com essas duas credenciais, é possível tentar abusar da porta RDP aberta no domínio. Primeiro com o usuário n[ENCONTRADO] falha, mas com o usuário b[ENCONTRADO] o acesso é realizado.

```bash
xfreerdp /v:lab.enterprise.thm /u:'nik' /p:'ToastyBoi!' /dynamic-resolution /clipboard /cert:ignore
#Retornou erro, não funcionando com o usuário do GitHub
xfreerdp /v:lab.enterprise.thm /u:'bitbucket' /p:'littleredbucket' /dynamic-resolution /clipboard /cert:ignore
#Acesso RDP com sucesso utilizando o usuário encontrado com o SPN.
```

<img width="1006" height="593" alt="image" src="https://github.com/user-attachments/assets/b0c34223-45de-43ca-85c3-2108fa4080b0" />

Assim, o arquivo user possui a primeira flag: THM{e[ENCONTRADO]}

---

> **Acesso obtido:** RDP estabelecido com o usuário e senha encontrado quebrando a hash SPN.

---

## 3. Escalada de Privilégios (Privilege Escalation)

### Enumeração Interna
Identificação do vetor de elevação de privilégios:

```bash
sudo -l
```

### Exploração e Obtenção de Root / SYSTEM
Explicação técnica da execução do bypass ou exploração do binário/permissão SUID.

```bash
# Comando utilizado para obter acesso privilegiado
```

> **Acesso obtido:** Acesso completo de administrador (`root` / `NT AUTHORITY\SYSTEM`).

---

## 4. Recomendações de Mitigação (Remediação)

1. **Aplicação Web:** Sanear e validar todas as entradas do usuário no backend antes do processamento.
2. **Hardening do SO:** Aplicar o princípio do menor privilégio, removendo permissões `sudo` desnecessárias para contas de serviço web.
