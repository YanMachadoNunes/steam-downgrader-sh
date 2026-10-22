# Steam Downgrader para Linux

Um script único para **voltar o cliente do Steam para a versão anterior** — ou para
qualquer versão antiga que você quiser — em Linux.

> **Atenção:** isto rebaixa o **programa Steam** (o cliente), **não** os jogos.
> Para rebaixar um jogo, o caminho é outro (console do Steam / DepotDownloader).

---

## Índice

- [Por que isso existe](#por-que-isso-existe)
- [Requisitos](#requisitos)
- [Instalação](#instalação)
- [Uso rápido (3 comandos)](#uso-rápido-3-comandos)
- [Todos os comandos](#todos-os-comandos)
- [Opções](#opções)
- [Exemplos](#exemplos)
- [Como funciona por dentro](#como-funciona-por-dentro)
- [Desfazendo (undo)](#desfazendo-undo)
- [Backup e recuperação deemergência](#backup-e-recuperação-de-emergência)
- [Problemas comuns](#problemas-comuns)
- [Avisos](#avisos)
- [Créditos](#créditos)

---

## Por que isso existe

Todo dia o Steam se atualiza sozinho em segundo plano. Quando isso acontece, o
cliente novo pode:

- quebrar a interface,
- travar ou abrir com tela preta,
- ficar lento,
- parar de funcionar em distros/OS mais antigos,
- dar conflito com overlays, mods ou ferramentas de terceiros.

O problema: **o servidor da Valve (CDN) só guarda a versão mais recente**.
Quando o cliente já atualizou, a versão antiga simplesmente não está mais lá
para baixar.

A solução usada aqui: o **Internet Archive** mantém um espelho de cada versão
do cliente já publicada, com os arquivos e o manifesto de cada uma. O script
baixa a versão desejada desse espelho, valida os arquivos por **SHA2** e
entrega o cliente antigo de volta para o Steam.

---

## Requisitos

- Linux
- `bash`, `curl`, `python3`, `unzip` e `sha256sum` (presentes na maioria das distros)
- Acesso à internet
- O Steam instalado (este script **não** instala o Steam)

Confira se está tudo aí:

```bash
for c in curl python3 unzip sha256sum; do command -v $c >/dev/null && echo "ok: $c" || echo "FALTA: $c"; done
```

---

## Instalação

```bash
git clone git@github.com:leomachadods/steam-downgrader-sh.git
cd steam-downgrader-sh
chmod +x steam-client-downgrade.sh
```

Não precisa instalar nada: o script é autocontido e roda direto na pasta.

---

## Uso rápido (3 comandos)

```bash
# 1. ver como está agora
./steam-client-downgrade.sh status

# 2. rebaixar o cliente para a versão anterior
./steam-client-downgrade.sh downgrade

# 3. voltar para a versão mais recente
./steam-client-downgrade.sh undo
```

O script **fecha o Steam sozinho** durante a operação e **abre de novo** no final.
Se algum jogo estiver aberto, feche-o antes.

---

## Todos os comandos

| Comando | O que faz |
|---|---|
| `status` | Mostra a versão instalada, a versão que está no servidor, se o auto-update está bloqueado, onde está o backup e se o Steam está rodando. |
| `start` | Apenas abre o Steam (útil se o script fechou e não voltou). |
| `list` | Lista todas as versões do cliente que estão arquivadas e disponíveis. |
| `downgrade [VERSAO]` | Rebaixa o cliente. Sem argumento, vai **uma versão atrás**. Com argumento, vai para a versão exata. |
| `undo` | Desfaz o rebaixamento e devolve o cliente mais recente. |
| `restore-backup` | Restaura o backup salvo do Steam (funciona sem internet). |
| `cleanup` | Apaga os backups salvos para liberar espaço. |
| `--help` | Mostra a ajuda. |

---

## Opções

| Opção | Efeito |
|---|---|
| `-y`, `--yes` | Não pergunta confirmação. |
| `-n`, `--dry-run` | Só mostra o que seria feito. **Não altera nada.** |
| `--steam-path DIR` | Caminho do Steam, caso a detecção automática falhe. |
| `--backup-dir DIR` | Onde guardar o backup. |
| `--no-restart` | Não reinicia o Steam no final. |

---

## Exemplos

```bash
# ver a situação atual
./steam-client-downgrade.sh status

# só ver o plano, sem mudar nada
./steam-client-downgrade.sh downgrade --dry-run

# rebaixar uma versão atrás (pede confirmação)
./steam-client-downgrade.sh downgrade

# rebaixar sem perguntar nada
./steam-client-downgrade.sh downgrade -y

# rebaixar para uma versão específica
./steam-client-downgrade.sh downgrade 1791918634

# ver o que está disponível
./steam-client-downgrade.sh list

# voltar para a versão mais recente
./steam-client-downgrade.sh undo -y

# voltar sem internet, a partir do backup
./steam-client-downgrade.sh restore-backup
```

---

## Como funciona por dentro

1. **Backup** — o script copia a pasta `package/` do Steam, o launcher e os
   arquivos de configuração para `~/steam-client-backup-<data>/`.
2. **Manifesto** — baixa o manifesto da versão desejada no Internet Archive.
3. **Validação** — baixa o launcher da versão antiga e confere o **SHA2** dele
   contra o hash registrado no manifesto. Se não bater, o script para.
4. **Bootstrap** — executa o Steam com
   `-forcesteamupdate -forcepackagedownload -overridepackageurl <arquivo>`
   para que o próprio Steam instale o pacote antigo.
5. **Trava do auto-update** — cria `~/.local/share/Steam/steam.cfg` com
   `BootStrapperInhibitAll=enable`, senão o cliente se atualiza de volta
   sozinha.
6. **Verificação** — o script reabre o Steam e confere no log qual versão o
   cliente realmente está reportando.

<details>
<summary>Detalhe técnico: por que existem dois pacotes</summary>

O cliente novo do Steam (a partir de 2026) distribui os arquivos em **dois
pacotes**: `steam_client_ubuntu12` e `steam_client_steamrt64`. Eles gravam
**nos mesmos caminhos** dentro da pasta do Steam, e cada launcher só sabe
cuidar de um deles.

Por isso o `undo` do script trabalha em duas fases:

1. usa o launcher antigo (o do pacote `ubuntu12`) para atualizar esse pacote no
   CDN da Valve;
2. usa o launcher novo (o do pacote `steamrt64`) para reinstalar esse pacote por
   último — assim os arquivos em disco terminam sendo os mais novos.

</details>

---

## Desfazendo (undo)

```bash
./steam-client-downgrade.sh undo
```

O que ele faz:

1. para o Steam e limpa locks obsoletos (`~/.steam/steam.pid` e `~/.steam/steam.pipe`);
2. atualiza o pacote `ubuntu12` direto do CDN da Valve;
3. reinstala o pacote `steamrt64` (é ele que escreve os arquivos do cliente);
4. remove o `BootStrapperInhibitAll` do `steam.cfg`, liberando o auto-update;
5. reabre o Steam e confere a versão.

---

## Backup e recuperação de emergência

Cada `downgrade` cria um backup automático em:

```
~/steam-client-backup-AAAAMMDD-HHMMSS/
```

Se algo der errado e você não tiver internet, restaure o backup:

```bash
./steam-client-downgrade.sh restore-backup
```

Para liberar espaço depois:

```bash
./steam-client-downgrade.sh cleanup
```

---

## Problemas comuns

**"Steam is already running"**
O launcher achou um `steam.pid` antigo de uma sessão encerrada à força.
O script já limpa isso automaticamente; se acontecer na mão:

```bash
rm -f ~/.steam/steam.pid ~/.steam/steam.pipe
```

**"Steam nao apareceu no processo"**
O Steam levou mais tempo para subir. Se a janela não abriu, rode
`./steam-client-downgrade.sh start` — ou simplesmente abra o Steam pelo seu
launcher usual.

**O cliente voltou sozinho para a versão nova**
O `steam.cfg` foi removido ou sobrescrito. Recrie:

```bash
echo "BootStrapperInhibitAll=enable" > ~/.local/share/Steam/steam.cfg
```

**A janela do Steam abre mas fica preta / crasha**
É o sintoma típico de cliente quebrado. Faça o `undo` e teste.

**Erro de rede ao baixar**
O Internet Archive pode estar lento ou limitado. Tente de novo em outro
momento, ou use `--dry-run` para confirmar que o restante do script está
funcionando.

---

## Avisos

- Isto **não** é suportado pela Valve. Use por sua conta e risco.
- O `steam.cfg` com `BootStrapperInhibitAll=enable` **bloqueia todas as
  atualizações do cliente** até você remover a linha. Para voltar a atualizar
  normalmente, apague o arquivo:
  ```bash
  rm ~/.local/share/Steam/steam.cfg
  ```
- Cliente muito antigo pode perder o login online com o passar do tempo (a
  Valve muda o servidor junto com o cliente). Se isso acontecer, ainda dá para
  usar o Steam em **modo offline** com os jogos instalados.
- O script fecha o Steam durante a operação. Salve seu trabalho antes.

---

## Créditos

- O método de rebaixamento vem do guia
  [Steam Client Downgrades & Survival Kit](https://steamcommunity.com/sharedfiles/filedetails?id=2967855237),
  do [lightwo](https://blog.lightwo.net/steam-client-downgrades-survival-kit.html/).
- Os arquivos das versões antigas vêm do espelho
  [`steam_steamcmd_packages`](https://archive.org/details/steam_steamcmd_packages)
  no Internet Archive.
- Testado no Linux com Steam `1791918634` ↔ `1792618285`.

## Licença

MIT — veja [LICENSE](LICENSE).
