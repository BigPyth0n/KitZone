# پروژه پلتفرم تحلیل داده‌های کندل (Kandel Analytics Platform)

این پروژه یک پلتفرم کامل برای جمع‌آوری، ذخیره‌سازی، تحلیل و مصورسازی داده‌های کندل بازارهای مالی (فیوچرز) است. معماری پروژه بر اساس میکروسرویس‌ها و کاملاً داکرایز شده است تا توسعه، استقرار و مدیریت آن به ساده‌ترین شکل ممکن انجام شود.

---

## 🏗️ معماری سیستم

معماری این پلتفرم از چند سرویس مستقل تشکیل شده که از طریق یک شبکه داخلی داکر با یکدیگر در ارتباط هستند و توسط یک Reverse Proxy مدیریت می‌شوند.

![Architecture Diagram](https://i.imgur.com/39w8o0x.png)

---

## 🛠️ پشته فناوری (Technology Stack)

* **بک‌اند و جمع‌آوری داده:** Python 3.10+, **FastAPI**
* **اپلیکیشن اصلی (SaaS):** **Anvil** (Full-stack Python)
* **دیتابیس:** **PostgreSQL** 16
* **مصورسازی و داشبورد:** **Apache Superset**
* **مدیریت دیتابیس:** **pgAdmin 4**
* **زیرساخت و کانتینرسازی:** **Docker** & **Docker Compose**
* **وب سرور و پروکسی معکوس:** **Nginx Proxy Manager**

---

## 🗺️ نقشه راه پروژه (Roadmap)

پروژه در سه فاز اصلی توسعه داده می‌شود:

### فاز ۱: هسته جمع‌آوری و ذخیره‌سازی داده (Data Ingestion Core)

**هدف:** ساخت یک سرویس پایدار برای دریافت داده‌های کندل از صرافی و ذخیره آن‌ها در دیتابیس.

#### مراحل اجرایی:

1.  **ایجاد ساختار پوشه:**
    ```bash
    mkdir data_ingestor
    cd data_ingestor
    ```

2.  **ایجاد فایل‌های سرویس FastAPI:**
    * `main.py`: فایل اصلی اپلیکیشن FastAPI.
    * `database.py`: منطق اتصال به دیتابیس.
    * `collector.py`: منطق دریافت داده از صرافی.
    * `requirements.txt`: لیست پکیج‌های پایتون.
    * `Dockerfile`: دستورالعمل ساخت ایمیج داکر.

3.  **نوشتن کدها:**
    * **`requirements.txt`**:
        ```txt
        fastapi
        uvicorn[standard]
        psycopg2-binary
        requests
        schedule
        ```
    * **`database.py`**:
        ```python
        import psycopg2
        import os

        def get_db_connection():
            conn = psycopg2.connect(
                host="postgres_db", # نام سرویس دیتابیس در داکر
                port="5432",
                user=os.environ.get("POSTGRES_USER"),
                password=os.environ.get("POSTGRES_PASSWORD"),
                dbname="postgres"
            )
            return conn
        ```
    * **`main.py`** (مثال ساده):
        ```python
        from fastapi import FastAPI
        import time
        # منطق collector و database را اینجا import کنید

        app = FastAPI(title="Data Ingestor Service")

        @app.get("/")
        def read_root():
            return {"status": "Data Ingestor is running"}

        # می‌توانید از Background Tasks برای اجرای دائمی collector استفاده کنید
        ```

4.  **ساخت `Dockerfile`:**
    ```dockerfile
    FROM python:3.10-slim

    WORKDIR /app

    COPY requirements.txt .
    RUN pip install --no-cache-dir -r requirements.txt

    COPY . .

    CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
    ```

5.  **اضافه کردن سرویس به `docker-compose.yml`:**
    ```yaml
    # ... (سرویس‌های postgres_db و pgadmin)
      data_ingestor:
        build: ./data_ingestor # مسیر پوشه سرویس
        container_name: data_ingestor_service
        restart: unless-stopped
        environment:
          # این متغیرها از Portainer یا فایل .env خوانده می‌شوند
          POSTGRES_USER: ${POSTGRES_USER}
          POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
        networks:
          - my_project_network
        depends_on:
          postgres_db:
            condition: service_healthy
    ```

---

### فاز ۲: مصورسازی و تحلیل داده‌ها (Dashboarding)

**هدف:** ایجاد داشبوردهای حرفه‌ای برای مشاهده داده‌ها با استفاده از Apache Superset.

#### مراحل اجرایی:

1.  **اضافه کردن Superset به `docker-compose.yml`:**
    ```yaml
    # ... (سایر سرویس‌ها)
      superset:
        image: apache/superset:latest
        container_name: superset_service
        environment:
          ADMIN_USERNAME: admin # نام کاربری ادمین سوپرست
          ADMIN_EMAIL: your_superset_admin@email.com
          ADMIN_PASSWORD: your_strong_password
          SUPERSET_SECRET_KEY: "your_super_secret_key_for_superset"
        ports:
          - "1050:8088" # پورت بیرونی برای دسترسی به سوپرست
        volumes:
          - superset_data:/app/superset_home
        networks:
          - my_project_network
        depends_on:
          postgres_db:
            condition: service_healthy
    ```
    > **نکته:** والیوم `superset_data` را هم به لیست `volumes` در انتهای فایل اضافه کنید.

2.  **اتصال Superset به دیتابیس:**
    * وارد محیط وب Superset شوید (مثلاً `http://<IP_سرور>:1050`).
    * به بخش **Data > Databases** بروید و روی **+ Database** کلیک کنید.
    * PostgreSQL را انتخاب کرده و از **SQLAlchemy URI** زیر برای اتصال استفاده کنید:
        ```
        postgresql+psycopg2://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres_db:5432/postgres
        ```
        > مقادیر متغیرها را با اطلاعات واقعی خود جایگزین کنید.

3.  **ساخت نمودار و داشبورد:**
    * از بخش **Datasets** جدول `futures_candles` را اضافه کنید.
    * به بخش **Charts** بروید و با استفاده از دیتاست جدید، نمودارهای مورد نظر خود را بسازید.
    * نمودارها را در یک **Dashboard** جدید کنار هم قرار دهید.

---

### فاز ۳: ساخت اپلیکیشن نهایی (SaaS Application)

**هدف:** ایجاد یک پلتفرم کامل با قابلیت ثبت‌نام، ورود و مدیریت اشتراک کاربران.

#### مراحل اجرایی:

1.  **توسعه اپلیکیشن در Anvil:**
    * پروژه خود را در پلتفرم Anvil بسازید. Anvil میزبانی اپلیکیشن شما را بر عهده می‌گیرد.

2.  **پیاده‌سازی مدیریت کاربران:**
    * از سرویس داخلی Anvil به نام `anvil.users` برای فرم‌های ثبت‌نام، ورود و مدیریت جلسات کاربران استفاده کنید.
    ```python
    # مثال برای فرم لاگین در Anvil
    anvil.users.login_with_form()
    ```

3.  **اتصال به دیتابیس از Anvil:**
    * در `Server Modules` در Anvil، یک تابع برای اتصال به دیتابیس PostgreSQL روی سرور خود بنویسید.
    > **هشدار:** اطلاعات اتصال (نام کاربری، رمز عبور، IP سرور) را حتماً در بخش **Secrets** در Anvil ذخیره کنید.
    ```python
    # کد در یک Server Module در Anvil
    import anvil.server
    import psycopg2

    @anvil.server.callable
    def get_user_data():
        # اطلاعات از Secrets خوانده می‌شود
        db_password = anvil.secrets.get_secret("DB_PASSWORD")
        conn = psycopg2.connect(
            host="<IP_عمومی_سرور_شما>",
            port="1040", # پورت بیرونی دیتابیس
            user="MasterDatabaseUser",
            password=db_password,
            dbname="postgres"
        )
        # ... ادامه کد برای خواندن داده‌ها
    ```
4.  **راه اندازی و نصب (Setup)**
    برای راه‌اندازی این پروژه به صورت محلی یا روی سرور:
     * ابتدا `docker` و `docker-compose` را نصب کنید
     * سپس یک فایل `.env` مانند نمونه زیر ایجاد کنید.
        ```
        # --- PostgreSQL Credentials ---
        POSTGRES_USER=MasterDatabaseUser
        POSTGRES_PASSWORD=7lJo2sCTK6l6wy

        # --- pgAdmin Credentials ---
        PGADMIN_EMAIL=behrouz.mokhber@gmail.com
        PGADMIN_PASSWORD=PKu4eKYaJnsREI

        # --- Resource Limits ---
        # محدودیت استفاده از CPU (مثلا 50% از یک هسته)
        CPU_LIMIT=0.50
        # محدودیت حافظه (مثلا 512 مگابایت)
        MEMORY_LIMIT=512M
        ```
      * در نهایت با دستور `docker-compose up -d` تمام سرویس ها را اجرا کنید.
