import streamlit as st
import gspread
from oauth2client.service_account import ServiceAccountCredentials
from datetime import datetime
import json
import os

# ========== ПОДКЛЮЧЕНИЕ К GOOGLE SHEETS ==========
def get_gsheet_client():
    # Для локального запуска — из файла
    if os.path.exists("credentials.json"):
        scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
        creds = ServiceAccountCredentials.from_json_keyfile_name("credentials.json", scope)
        return gspread.authorize(creds)
    # Для Streamlit Cloud — из секретов
    else:
        creds_dict = dict(st.secrets["gcp_service_account"])
        scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
        creds = ServiceAccountCredentials.from_json_keyfile_dict(creds_dict, scope)
        return gspread.authorize(creds)

# ID твоей таблицы (ЗАМЕНИ НА СВОЙ!)
SHEET_ID = "1ABC123xyz"  # <--- СЮДА ВСТАВЬ ID СВОЕЙ ТАБЛИЦЫ

def get_fines_sheet():
    client = get_gsheet_client()
    return client.open_by_key(SHEET_ID).worksheet("fines")

def get_users_sheet():
    client = get_gsheet_client()
    return client.open_by_key(SHEET_ID).worksheet("users")

# ========== ФУНКЦИИ ДЛЯ РАБОТЫ С ДАННЫМИ ==========
def load_users():
    sheet = get_users_sheet()
    data = sheet.get_all_values()
    if len(data) <= 1:
        return []
    users = []
    for row in data[1:]:
        if row and row[0]:
            users.append({
                'badge': row[0],
                'phone': row[1],
                'name': row[2] if len(row) > 2 else f'Сотрудник {row[0]}'
            })
    return users

def save_user(badge, phone, name):
    sheet = get_users_sheet()
    sheet.append_row([badge, phone, name])

def load_fines():
    sheet = get_fines_sheet()
    data = sheet.get_all_values()
    if len(data) <= 1:
        return []
    fines = []
    for row in data[1:]:
        if row and row[0]:
            fines.append({
                'id': int(row[0]) if row[0].isdigit() else len(fines) + 1,
                'badge': row[1],
                'fine_date': row[2],
                'description': row[3],
                'comment': row[4],
                'admin_answer': row[5] if len(row) > 5 else '',
                'created_at': row[6] if len(row) > 6 else '',
                'status': row[7] if len(row) > 7 else 'new'
            })
    return fines

def save_fine(fine):
    sheet = get_fines_sheet()
    sheet.append_row([
        fine['id'],
        fine['badge'],
        fine['fine_date'],
        fine['description'],
        fine['comment'],
        fine['admin_answer'],
        fine['created_at'],
        fine['status']
    ])

def update_fine_answer(fine_id, answer_text):
    sheet = get_fines_sheet()
    data = sheet.get_all_values()
    for i, row in enumerate(data):
        if i == 0:
            continue
        if row and row[0] == str(fine_id):
            sheet.update_cell(i + 1, 6, answer_text)
            sheet.update_cell(i + 1, 8, 'answered')
            return True
    return False

# ========== ИНИЦИАЛИЗАЦИЯ СЕССИИ ==========
if 'fines' not in st.session_state:
    st.session_state.fines = load_fines()
if 'users' not in st.session_state:
    st.session_state.users = load_users()

# ========== СТРАНИЦА ВХОДА ==========
def login_page():
    st.title("🔐 Вход в систему учёта штрафов")
    
    tab1, tab2 = st.tabs(["👤 Сотрудник", "👑 Администратор"])
    
    with tab1:
        badge = st.text_input("Номер бейджа", key="badge_user")
        phone = st.text_input("Номер телефона", key="phone_user")
        if st.button("Войти как сотрудник"):
            user = next((u for u in st.session_state.users if u['badge'] == badge), None)
            if user:
                if user['phone'] == phone:
                    st.session_state['user'] = user
                    st.session_state['logged_in'] = True
                    st.rerun()
                else:
                    st.error("Неверный телефон")
            else:
                # Регистрация нового
                name = f'Сотрудник {badge}'
                new_user = {'badge': badge, 'phone': phone, 'name': name}
                st.session_state.users.append(new_user)
                save_user(badge, phone, name)
                st.session_state['user'] = new_user
                st.session_state['logged_in'] = True
                st.rerun()
    
    with tab2:
        admin_login = st.text_input("Логин", key="admin_login")
        admin_pass = st.text_input("Пароль", type="password", key="admin_pass")
        if st.button("Войти как администратор"):
            if admin_login == "admin" and admin_pass == "12345":
                st.session_state['user'] = {'badge': 'ADMIN', 'name': 'Администратор'}
                st.session_state['logged_in'] = True
                st.rerun()
            else:
                st.error("Неверные данные")

# ========== ДАШБОРД СОТРУДНИКА ==========
def user_dashboard(user):
    st.title(f"📋 Панель сотрудника {user['badge']}")
    
    with st.expander("➕ Добавить штраф", expanded=True):
        with st.form("add_fine"):
            fine_date = st.date_input("Дата", datetime.now())
            description = st.text_input("Описание")
            comment = st.text_area("Комментарий")
            submitted = st.form_submit_button("Сохранить")
            if submitted and description and comment:
                fines = load_fines()
                next_id = max([f['id'] for f in fines] + [0]) + 1
                new_fine = {
                    'id': next_id,
                    'badge': user['badge'],
                    'fine_date': fine_date.strftime('%Y-%m-%d'),
                    'description': description,
                    'comment': comment,
                    'admin_answer': '',
                    'created_at': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                    'status': 'new'
                }
                save_fine(new_fine)
                st.session_state.fines = load_fines()
                st.success("✅ Штраф добавлен!")
                st.rerun()
    
    st.subheader("📜 История штрафов")
    user_fines = [f for f in st.session_state.fines if f['badge'] == user['badge']]
    if not user_fines:
        st.info("Нет штрафов")
    else:
        for f in user_fines:
            with st.container():
                st.markdown(f"**{f['fine_date']} — {f['description']}**")
                st.caption(f"Комментарий: {f['comment']}")
                if f['admin_answer']:
                    st.success(f"📨 Ответ: {f['admin_answer']}")
                else:
                    st.caption("⏳ Ожидает ответа")
                st.divider()

# ========== ДАШБОРД АДМИНА ==========
def admin_dashboard():
    st.title("👑 Админ-панель")
    st.subheader("Все штрафы")
    
    st.session_state.fines = load_fines()
    
    if not st.session_state.fines:
        st.info("Нет штрафов")
    else:
        for f in st.session_state.fines:
            with st.container():
                col1, col2 = st.columns([3, 1])
                with col1:
                    st.markdown(f"**{f['badge']}** | {f['fine_date']} — {f['description']}")
                    st.caption(f"Комментарий: {f['comment']}")
                    if f['admin_answer']:
                        st.success(f"📨 Ответ: {f['admin_answer']}")
                    else:
                        st.warning("⏳ Ожидает ответа")
                with col2:
                    if not f['admin_answer']:
                        answer = st.text_input("Ответ", key=f"ans_{f['id']}", placeholder="Напишите...")
                        if st.button("Отправить", key=f"btn_{f['id']}"):
                            update_fine_answer(f['id'], answer)
                            st.session_state.fines = load_fines()
                            st.rerun()
                st.divider()

# ========== ОСНОВНАЯ ЛОГИКА ==========
if 'logged_in' not in st.session_state:
    st.session_state.logged_in = False

if not st.session_state.logged_in:
    login_page()
else:
    if st.sidebar.button("Выйти"):
        st.session_state.logged_in = False
        st.session_state.user = None
        st.rerun()
    
    if st.session_state.user['badge'] == 'ADMIN':
        admin_dashboard()
    else:
        user_dashboard(st.session_state.user)
