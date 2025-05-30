import os
import logging
from dotenv import load_dotenv
load_dotenv()
from telegram import Update, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.ext import Application, CommandHandler, MessageHandler, filters, ContextTypes, ConversationHandler, CallbackQueryHandler
from datetime import datetime, time
import pytz
import json

# Configuração de Logging
logging.basicConfig(format="%(asctime)s - %(name)s - %(levelname)s - %(message)s", level=logging.INFO)
logger = logging.getLogger(__name__)

# Carregar o token da variável de ambiente
TOKEN = os.environ.get("TELEGRAM_BOT_TOKEN")
if not TOKEN:
    logger.error("Erro: A variável de ambiente TELEGRAM_BOT_TOKEN não está configurada.")
    exit() # Ou raise ValueError("Token não encontrado")

DATA_FILE = "ru_credits_data.json"
MAX_CREDITS_PER_MEAL = 22

# Fuso horário de referência para as refeições
TIMEZONE = pytz.timezone("America/Sao_Paulo") # Exemplo, ajuste se necessário

# Janelas de tempo para as refeições (horas locais)
MEAL_TIMES = {
    "cafe": (time(7, 0), time(9, 0)),
    "almoco": (time(11, 30), time(13, 0)), # Original era 13h, mas range é exclusivo no final, então 13:00:01 já não pega.
    "jantar": (time(17, 30), time(19, 0)) # Mesma lógica aqui.
}
# Para fins de comparação, usamos 13:00 e 19:00 como limites *exclusivos* superiores.
# Ou seja, 13:00:00 ainda é almoço, 13:00:01 não é.
# Na prática, time(X, Y) significa que vai até X:Y:00. Para incluir 13:00, o range deve ir até um pouco depois, ou
# a comparação deve ser <=. Vamos usar a lógica de que 13:00 é o último momento.

# Estados para ConversationHandler (Adicionar Créditos)
CHOOSE_MEAL_ADD, CHOOSE_QUANTITY_ADD = range(2)

# ...
# Caminho para o disco persistente no Render
RENDER_DISK_MOUNT_PATH = "/var/data" # Você definirá isso nas configurações do Disco no Render
DATA_FILE_NAME = "ru_credits_data.json"
DATA_FILE = os.path.join(RENDER_DISK_MOUNT_PATH, DATA_FILE_NAME)

# Criar o diretório de montagem do disco se não existir (importante na primeira execução)
try:
    os.makedirs(RENDER_DISK_MOUNT_PATH, exist_ok=True)
    logger.info(f"Diretório do disco {RENDER_DISK_MOUNT_PATH} verificado/criado.")
except OSError as e:
    logger.error(f"Erro ao criar o diretório do disco {RENDER_DISK_MOUNT_PATH}: {e}")
    # Considerar tratamento de erro mais robusto se necessário

def load_data():
    if os.path.exists(DATA_FILE):
        try:
            with open(DATA_FILE, 'r') as f:
                return json.load(f)
        except json.JSONDecodeError:
            logger.warning(f"Arquivo {DATA_FILE} corrompido ou vazio. Retornando dados vazios.")
            return {}
        except Exception as e:
            logger.error(f"Erro ao carregar {DATA_FILE}: {e}")
            return {}
    logger.info(f"Arquivo {DATA_FILE} não encontrado. Retornando dados vazios (será criado no primeiro save).")
    return {}

def save_data(data):
    try:
        with open(DATA_FILE, 'w') as f:
            json.dump(data, f, indent=2)
        logger.info(f"Dados salvos em {DATA_FILE}")
    except Exception as e:
        logger.error(f"Erro ao salvar dados em {DATA_FILE}: {e}")
# ... resto do seu código ...
# ... (resto do seu código: get_user_data, start_command, etc.)
# --- Funções Auxiliares ---
def get_current_meal_type():
    now_local = datetime.now(TIMEZONE).time()
    for meal, (start_time, end_time) in MEAL_TIMES.items():
        # Se a hora final for antes da inicial (ex: noturno passando da meia-noite, não é o caso aqui)
        # if end_time < start_time:
        #     if now_local >= start_time or now_local < end_time: #  <= end_time para incluir o horário final exato
        #         return meal
        # else:
        if start_time <= now_local < end_time: # < end_time para ser até HH:MM:59
            return meal
        # Ajuste para incluir o horário final exato:
        # Para o almoço (11:30-13:00), queremos que 13:00:00 seja incluído.
        # Para o jantar (17:30-19:00), queremos que 19:00:00 seja incluído.
        # A forma mais simples é ajustar a lógica de comparação ou o `end_time`
        # Para este exemplo, vamos considerar que o `end_time` é exclusivo.
        # Se for para ser inclusivo (até 13:00:00 e 19:00:00), a lógica seria:
        # if meal == "almoco" and start_time <= now_local <= end_time:
        #    return meal
        # if meal == "jantar" and start_time <= now_local <= end_time:
        #    return meal
        # if meal == "cafe" and start_time <= now_local < end_time: # Café é até 8:59:59
        #    return meal
    return None # Fora do horário de qualquer refeição

# --- Handlers de Comando ---
async def start_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id_str = str(update.effective_user.id)
    get_user_data(user_id_str) # Garante que o usuário seja inicializado
    save_data(load_data()) # Salva se foi criado um novo usuário

    text = (
        "Bem-vindo ao Bot de Créditos do RU!\n\n"
        "Comandos disponíveis:\n"
        "/saldo - Verifica seus créditos.\n"
        "/adicionar - Adiciona créditos para uma refeição.\n"
        "/usar - Registra o consumo de uma refeição (automático pelo horário).\n"
        "/ajuda - Mostra esta mensagem."
    )
    await update.message.reply_text(text)

async def saldo_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id_str = str(update.effective_user.id)
    user_credits = get_user_data(user_id_str)

    text = (
        "Seu saldo de créditos:\n"
        f"☕ Café da Manhã: {user_credits.get('cafe', 0)}\n"
        f"🍲 Almoço: {user_credits.get('almoco', 0)}\n"
        f"🍛 Jantar: {user_credits.get('jantar', 0)}"
    )
    await update.message.reply_text(text)

async def usar_credito_command(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id_str = str(update.effective_user.id)
    user_data = get_user_data(user_id_str)
    all_data = load_data()

    meal_type = get_current_meal_type()
    today_str = datetime.now(TIMEZONE).strftime("%Y-%m-%d")

    if not meal_type:
        await update.message.reply_text("Desculpe, não há nenhuma refeição disponível para consumo neste horário.")
        return

    # Mapeamento para nomes mais amigáveis
    meal_name_map = {"cafe": "Café da Manhã", "almoco": "Almoço", "jantar": "Jantar"}
    friendly_meal_name = meal_name_map.get(meal_type, meal_type.capitalize())

    last_consumed_key = f"ultimo_consumo_{meal_type}"

    if user_data.get(last_consumed_key) == today_str:
        await update.message.reply_text(f"Você já consumiu {friendly_meal_name} hoje.")
        return

    if user_data.get(meal_type, 0) > 0:
        user_data[meal_type] -= 1
        user_data[last_consumed_key] = today_str
        all_data[user_id_str] = user_data # Atualiza no dicionário principal
        save_data(all_data)
        await update.message.reply_text(
            f"1 crédito de {friendly_meal_name} consumido com sucesso!\n"
            f"Saldo restante de {friendly_meal_name}: {user_data[meal_type]}"
        )
    else:
        await update.message.reply_text(f"Saldo de {friendly_meal_name} insuficiente.")

# --- ConversationHandler para Adicionar Créditos ---
async def adicionar_start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    keyboard = [
        [InlineKeyboardButton("☕ Café da Manhã", callback_data="cafe")],
        [InlineKeyboardButton("🍲 Almoço", callback_data="almoco")],
        [InlineKeyboardButton("🍛 Jantar", callback_data="jantar")],
        [InlineKeyboardButton("❌ Cancelar", callback_data="cancel_add")],
    ]
    reply_markup = InlineKeyboardMarkup(keyboard)
    await update.message.reply_text("Qual refeição você gostaria de adicionar créditos?", reply_markup=reply_markup)
    return CHOOSE_MEAL_ADD

async def choose_meal_add(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    await query.answer() # Responde ao callback para remover o "loading" no botão

    meal_choice = query.data
    if meal_choice == "cancel_add":
        await query.edit_message_text("Operação cancelada.")
        return ConversationHandler.END

    context.user_data['meal_to_add'] = meal_choice # Salva a escolha da refeição

    # Mapeamento para nomes mais amigáveis
    meal_name_map = {"cafe": "Café da Manhã", "almoco": "Almoço", "jantar": "Jantar"}
    friendly_meal_name = meal_name_map.get(meal_choice, meal_choice.capitalize())

    await query.edit_message_text(f"Você selecionou {friendly_meal_name}. Quantos créditos deseja adicionar (1-{MAX_CREDITS_PER_MEAL})?")
    return CHOOSE_QUANTITY_ADD

async def choose_quantity_add(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id_str = str(update.effective_user.id)
    user_data = get_user_data(user_id_str)
    all_data = load_data()
    meal_type = context.user_data.get('meal_to_add')

    try:
        quantity = int(update.message.text)
        if not (1 <= quantity <= MAX_CREDITS_PER_MEAL):
            await update.message.reply_text(f"Por favor, insira um número entre 1 e {MAX_CREDITS_PER_MEAL}.")
            # Volta para o mesmo estado para o usuário tentar novamente
            return CHOOSE_QUANTITY_ADD
    except ValueError:
        await update.message.reply_text("Entrada inválida. Por favor, insira um número.")
        return CHOOSE_QUANTITY_ADD

    current_credits = user_data.get(meal_type, 0)
    new_total_credits = current_credits + quantity

    credits_to_add = quantity
    if new_total_credits > MAX_CREDITS_PER_MEAL:
        credits_to_add = MAX_CREDITS_PER_MEAL - current_credits # Adiciona apenas o que falta para o máximo
        if credits_to_add < 0: credits_to_add = 0 # Caso já esteja no máximo ou acima (não deveria)
        user_data[meal_type] = MAX_CREDITS_PER_MEAL
        message = (
            f"Você tentou adicionar {quantity}, mas o limite é {MAX_CREDITS_PER_MEAL}. "
            f"{credits_to_add} créditos de {meal_type} foram adicionados. "
            f"Seu saldo de {meal_type} agora é {MAX_CREDITS_PER_MEAL}."
        )
    else:
        user_data[meal_type] = new_total_credits
        message = (
            f"{quantity} créditos de {meal_type} adicionados com sucesso! "
            f"Seu novo saldo de {meal_type} é {user_data[meal_type]}."
        )

    all_data[user_id_str] = user_data
    save_data(all_data)
    await update.message.reply_text(message)

    context.user_data.pop('meal_to_add', None) # Limpa o dado da conversa
    return ConversationHandler.END

async def cancel_add(update: Update, context: ContextTypes.DEFAULT_TYPE):
    query = update.callback_query
    if query: # Se cancelado por botão inline
        await query.answer()
        await query.edit_message_text("Adição de créditos cancelada.")
    else: # Se cancelado por comando /cancel (não implementado aqui, mas boa prática)
        await update.message.reply_text("Adição de créditos cancelada.")

    context.user_data.pop('meal_to_add', None)
    return ConversationHandler.END

# --- Função Principal ---
def main():
    application = Application.builder().token(TOKEN).build()

    # ConversationHandler para adicionar créditos
    add_conv_handler = ConversationHandler(
        entry_points=[CommandHandler("adicionar", adicionar_start)],
        states={
            CHOOSE_MEAL_ADD: [CallbackQueryHandler(choose_meal_add)],
            CHOOSE_QUANTITY_ADD: [MessageHandler(filters.TEXT & ~filters.COMMAND, choose_quantity_add)],
        },
        fallbacks=[
            CallbackQueryHandler(cancel_add, pattern="^cancel_add$"),
            CommandHandler("cancelar_adicao", cancel_add) # Comando para cancelar
        ],
    )

    application.add_handler(CommandHandler("start", start_command))
    application.add_handler(CommandHandler("ajuda", start_command)) # Alias para /start
    application.add_handler(CommandHandler("saldo", saldo_command))
    application.add_handler(CommandHandler("usar", usar_credito_command)) # ou /consumir
    application.add_handler(add_conv_handler)

    logger.info("Bot iniciado. Pressione Ctrl+C para parar.")
    application.run_polling()

if __name__ == "__main__":
    main()
