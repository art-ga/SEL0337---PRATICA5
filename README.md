## ANTONIO ROSA GALINDO DE ANDRADE 15440868
## GUSTAVO BREDA COELHO 14746742

# Sistema Embarcado Linux: Controle GPIO com Python e Gerenciamento via `systemd`

Projeto prático de Sistemas Embarcados desenvolvido para **Raspberry Pi (Linux Embarcado)** utilizando **Python (`gpiozero`)** e gerenciamento de ciclo de vida de processos via **`systemd`**.

A aplicação executa automaticamente durante o boot do sistema operacional (sem necessidade de login manual ou scripts `.sh` intermediários), sinalizando o estado de operação por meio de LEDs físicos e executando uma rotina dedicada de encerramento quando o serviço é interrompido via `systemctl stop`.

---

## Estrutura do Repositório

```text
projeto-embarcados/
├── README.md            # Documentação completa de arquitetura, hardware e execução
├── monitor_led.py       # Script principal executado no boot (ExecStart)
├── parada_led.py        # Script executado na parada do serviço (ExecStop)
└── monitor_led.service  # Arquivo de unidade (Unit File) do systemd
```

---

## 1. Arquitetura e Funcionamento

### Chamada Direta ao Interpretador Python no Unit File
Diferente de binários compilados em C/C++, um arquivo `.py` não é executado diretamente pelo kernel Linux de forma isolada: ele depende do **interpretador Python (`/usr/bin/python3`)** para carregar o ambiente de execução e suas bibliotecas de acesso ao hardware (`gpiozero` e `lgpio`).

Por esse motivo, no arquivo `monitor_led.service`, o próprio interpretador Python é invocado explicitamente tanto na inicialização (`ExecStart`) quanto na parada (`ExecStop`), dispensando encapsulamentos em shell script (`script.sh`):
* **`ExecStart=/usr/bin/python3 .../monitor_led.py`**: Carrega o interpretador Python no boot, faz o *blink* contínuo do LED Verde (indicando serviço ativo) e monitora tanto um botão físico quanto a temperatura da CPU via `sysfs` (`/sys/class/thermal/thermal_zone0/temp`).
* **`ExecStop=/usr/bin/python3 .../parada_led.py`**: Disparado automaticamente pelo `systemd` ao executar `sudo systemctl stop monitor_led.service`. Invoca uma segunda rotina em Python que pisca o LED Vermelho 3 vezes para confirmar visualmente a parada do serviço e registra o evento no log do sistema (`journald`).

### `systemd` vs. `SystemV`

| Característica | SystemV (`init`) | `systemd` (Adotado neste projeto) |
| :--- | :--- | :--- |
| **Inicialização** | Sequencial por meio de scripts shell em `/etc/init.d/` | Paralela baseada em dependências e alvos (`targets`) |
| **Configuração** | Scripts Bash complexos com códigos de retorno manuais | Arquivos declarativos (*Unit Files* `.service`) |
| **Monitoramento** | Sem supervisão nativa de queda de processos | Supervisão nativa (`Restart=`) e logs centralizados (`journalctl`) |
| **Gerenciamento** | Comandos `service <nome> start/stop` | Utilitário unificado `systemctl` |

---

## 2. Montagem do Hardware (Protoboard ou Simulador)

### Lista de Componentes
* **1x** Raspberry Pi (3B+, 4B, 5 ou Zero 2 W) com Raspberry Pi OS.
* **1x** Protoboard.
* **1x** LED Verde 5mm (Indicador de atividade / *Blink* no boot).
* **1x** LED Vermelho 5mm (Indicador de alerta térmico, teste via botão e sinalização de parada).
* **2x** Resistores de $220\,\Omega$ ou $330\,\Omega$.
* **1x** Push Button (chave táctil).
* **Jumpers** Macho-Fêmea e Macho-Macho.

### Tabela de Conexões (Pinout)

| Componente | Terminal | Pino BCM | Pino Físico (Header) | Função |
| :--- | :--- | :--- | :--- | :--- |
| **Barramento Comum** | Negativo (`-`) | `GND` | **Pino 6** | Referência terra na protoboard |
| **LED Verde** | Ânodo (`+`) via $220\,\Omega$ | `GPIO 17` | **Pino 11** | *Blink* contínuo enquanto o serviço roda |
| **LED Vermelho** | Ânodo (`+`) via $220\,\Omega$ | `GPIO 27` | **Pino 13** | Alerta/Botão e aviso visual no `ExecStop` |
| **Push Button** | Terminal A (Term. B no `GND`) | `GPIO 22` | **Pino 15** | Acionamento manual para demonstração |

### Diagrama Esquemático

```text
Raspberry Pi (Header 40 pinos)               Protoboard
─────────────────────────────────            ───────────────────────────────
Pino 6  (GND)     ─────────────────────────► Barramento Negativo (-)
Pino 11 (GPIO 17) ──► [ Resistor 220 Ω ] ──► (+) LED Verde (-)     ──► GND
Pino 13 (GPIO 27) ──► [ Resistor 220 Ω ] ──► (+) LED Vermelho (-)  ──► GND
Pino 15 (GPIO 22) ─────────────────────────► [ Push Button ]       ──► GND
```

> **Execução em Simulador:** Caso utilize o **Proteus VSM** ou **Cirkit Designer**, conecte os componentes exatamente aos pinos físicos indicados acima. Para validar apenas a lógica em terminal Linux sem placa física, utilize `GPIOZERO_PIN_FACTORY=mock /usr/bin/python3 monitor_led.py`.

---

## 3. Instalação, Ajuste de Usuário e Validação (`systemctl`)

Para validar o projeto em qualquer máquina Linux/Raspberry Pi independentemente do nome de usuário configurado no sistema, siga os passos abaixo no terminal:

### Passo 1: Clonar o repositório e instalar dependências
```bash
sudo apt update && sudo apt install -y python3-gpiozero python3-lgpio
git clone <https://github.com/art-ga/SEL0337---PRATICA5.git> ~/projeto-embarcados
cd ~/projeto-embarcados
chmod +x monitor_led.py parada_led.py
```

### Passo 2: Ajustar o usuário no `.service` e registrar no `systemd`
Os comandos abaixo substituem automaticamente `pi` e `/home/pi/projeto-embarcados` pelo usuário atual (`$USER`) e diretório atual (`$PWD`):

```bash
sed -i "s|User=pi|User=$USER|g" monitor_led.service
sed -i "s|/home/pi/projeto-embarcados|$PWD|g" monitor_led.service

sudo cp monitor_led.service /etc/systemd/system/
sudo systemctl daemon-reload
```

### Passo 3: Habilitar no boot e validar com `systemctl`

1. **Habilitar inicialização automática no boot e iniciar o serviço:**
   ```bash
   sudo systemctl enable monitor_led.service
   sudo systemctl start monitor_led.service
   ```

2. **Verificar o status do serviço em execução:**
   ```bash
   systemctl status monitor_led.service
   ```
   *(Confirme o estado `active (running)` e observe o LED Verde piscando).*

3. **Testar a diretiva `ExecStop=` parando o serviço:**
   ```bash
   sudo systemctl stop monitor_led.service
   journalctl -u monitor_led.service -n 15 --no-pager
   ```
   *(Observe o LED Vermelho piscar 3 vezes e confira no log as mensagens `[MONITOR]` seguidas por `[EXECSTOP]`).*

4. **Testar persistência após reinicialização completa (Boot):**
   ```bash
   sudo systemctl start monitor_led.service
   sudo reboot
   ```
   *(Após o reinício, o LED Verde voltará a piscar automaticamente sem necessidade de login).*
