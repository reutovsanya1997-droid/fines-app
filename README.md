import streamlit as st
from datetime import datetime

# --- ИНИЦИАЛИЗАЦИЯ СЕССИИ ---
if 'fines' not in st.session_state:
    st.session_state.fines = []
if 'users' not in st.session_state:
    st.session_state.users = [{'badge': '123', 'phone': '111', 'name': 'Иван'}]

# --- СТРАНИЦА ВХОДА ---
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
                new_user = {'badge': badge, 'phone': phone, 'name': f'Сотрудник {badge}'}
                st.session_state.users.append(new_user)
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

# --- ДАШБОРД СОТРУДНИКА ---
def user_dashboard(user):
    st.title(f"📋 Панель сотрудника {user['badge']}")
    
    with st.expander("➕ Добавить штраф", expanded=True):
        with st.form("add_fine"):
            fine_date = st.date_input("Дата", datetime.now())
            description = st.text_input("Описание")
            comment = st.text_area("Комментарий")
            submitted = st.form_submit_button("Сохранить")
            if submitted and description and comment:
                new_fine = {
                    'id': len(st.session_state.fines) + 1,
                    'badge': user['badge'],
                    'fine_date': fine_date.strftime('%Y-%m-%d'),
                    'description': description,
                    'comment': comment,
                    'admin_answer': '',
                    'created_at': datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
                    'status': 'new'
                }
                st.session_state.fines.append(new_fine)
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

# --- ДАШБОРД АДМИНА ---
def admin_dashboard():
    st.title("👑 Админ-панель")
    st.subheader("Все штрафы")
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
                            f['admin_answer'] = answer
                            st.rerun()
                st.divider()

# --- ОСНОВНАЯ ЛОГИКА ---
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
