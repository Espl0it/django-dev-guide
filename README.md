# Django 开发指南

Django 是一个高级 Python Web 框架，鼓励快速开发和简洁实用的设计。

## 目录

1. [项目结构](#项目结构)
2. [环境配置](#环境配置)
3. [最佳实践](#最佳实践)
4. [模型设计](#模型设计)
5. [视图与路由](#视图与路由)
6. [API 开发](#api-开发)
7. [安全建议](#安全建议)
8. [部署指南](#部署指南)

---

## 项目结构

```
myproject/
├── myproject/           # 项目配置
│   ├── __init__.py
│   ├── settings.py      # 项目设置
│   ├── urls.py          # 根路由
│   ├── wsgi.py         # WSGI 入口
│   └── asgi.py         # ASGI 入口
├── apps/                # 应用目录
│   ├── users/          # 用户应用
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── views.py
│   │   ├── urls.py
│   │   ├── admin.py
│   │   ├── apps.py
│   │   ├── serializers.py  # DRF
│   │   └── tests.py
│   └── blog/           # 博客应用
├── templates/            # 模板目录
├── static/              # 静态文件
├── media/               # 用户上传
├── requirements.txt     # 依赖
├── .env                 # 环境变量
├── manage.py            # Django 管理脚本
└── pytest.ini          # 测试配置
```

---

## 环境配置

### 1. 虚拟环境

```bash
# 创建虚拟环境
python -m venv venv

# 激活
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows

# 安装依赖
pip install django djangorestframework psycopg2-binary python-dotenv
```

### 2. requirements.txt

```
Django>=4.2,<5.0
djangorestframework>=3.14,<4.0
psycopg2-binary>=2.9
python-dotenv>=1.0
celery>=5.3
redis>=4.5
gunicorn>=21.0
```

### 3. 环境变量 (.env)

```bash
# .env
DEBUG=True
SECRET_KEY=your-secret-key-here
ALLOWED_HOSTS=localhost,127.0.0.1
DATABASE_URL=postgres://user:pass@localhost:5432/mydb
REDIS_URL=redis://localhost:6379/0
```

### 4. settings.py 配置

```python
# settings.py
import os
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv('SECRET_KEY')

DEBUG = os.getenv('DEBUG', 'False') == 'True'

ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', '').split(',')

# 数据库
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST', 'localhost'),
        'PORT': os.getenv('DB_PORT', '5432'),
    }
}

# 应用
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'corsheaders',
    'apps.users',
]

# REST Framework
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 20,
}

# 静态文件
STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'

# 媒体文件
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

---

## 最佳实践

### 1. 项目初始化

```bash
# 创建项目
django-admin startproject myproject .

# 创建应用
python manage.py startapp users

# 迁移
python manage.py makemigrations
python manage.py migrate

# 超级用户
python manage.py createsuperuser

# 运行
python manage.py runserver
```

### 2. 代码规范

```python
# ✅ 正确：使用路径导入
from apps.users.models import User
from apps.blog.serializers import PostSerializer

# ❌ 错误：避免循环导入
# from blog.models import Post  # 在 users/models.py 中

# ✅ 正确：类型注解
from typing import Optional
from django.db.models import QuerySet

def get_user(pk: int) -> Optional[User]:
    return User.objects.filter(pk=pk).first()
```

### 3. 应用结构

```python
# apps/users/models.py
from django.db import models
from django.contrib.auth.models import AbstractUser


class User(AbstractUser):
    """自定义用户模型"""
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        db_table = 'users'
        ordering = ['-created_at']

    def __str__(self):
        return self.username
```

```python
# apps/users/apps.py
from django.apps import AppConfig


class UsersConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'apps.users'
    verbose_name = '用户'
```

```python
# settings.py 添加
AUTH_USER_MODEL = 'users.User'
```

---

## 模型设计

### 1. 基类模型

```python
# core/models.py
from django.db import models


class TimeStampedModel(models.Model):
    """时间戳基类"""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    class Meta:
        abstract = True


class UUIDModel(models.Model):
    """UUID主键基类"""
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)

    class Meta:
        abstract = True
```

### 2. 使用示例

```python
# apps/blog/models.py
from django.db import models
from django.conf import settings
from core.models import TimeStampedModel


class Category(TimeStampedModel):
    name = models.CharField(max_length=100, unique=True)
    slug = models.SlugField(max_length=100, unique=True)

    class Meta:
        verbose_name = '分类'
        verbose_name_plural = '分类'

    def __str__(self):
        return self.name


class Post(TimeStampedModel):
    title = models.CharField(max_length=200)
    slug = models.SlugField(max_length=200, unique=True)
    content = models.TextField()
    author = models.ForeignKey(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name='posts'
    )
    category = models.ForeignKey(
        Category,
        on_delete=models.SET_NULL,
        null=True,
        related_name='posts'
    )
    status = models.CharField(
        max_length=10,
        choices=[
            ('draft', '草稿'),
            ('published', '已发布'),
        ],
        default='draft'
    )
    views = models.PositiveIntegerField(default=0)

    class Meta:
        ordering = ['-created_at']
        indexes = [
            models.Index(fields=['-created_at']),
            models.Index(fields=['status', '-created_at']),
        ]

    def __str__(self):
        return self.title
```

---

## 视图与路由

### 1. REST API 视图

```python
# apps/users/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from .models import User
from .serializers import UserSerializer, UserCreateSerializer


class UserViewSet(viewsets.ModelViewSet):
    """用户视图集"""
    queryset = User.objects.all()
    serializer_class = UserSerializer
    permission_classes = [IsAuthenticated]

    def get_serializer_class(self):
        if self.action == 'create':
            return UserCreateSerializer
        return UserSerializer

    @action(detail=False, methods=['get'])
    def me(self, request):
        """当前用户信息"""
        serializer = self.get_serializer(request.user)
        return Response(serializer.data)

    @action(detail=False, methods=['post'])
    def change_password(self, request):
        """修改密码"""
        user = request.user
        old_password = request.data.get('old_password')
        new_password = request.data.get('new_password')

        if not user.check_password(old_password):
            return Response(
                {'error': '原密码错误'},
                status=status.HTTP_400_BAD_REQUEST
            )

        user.set_password(new_password)
        user.save()
        return Response({'message': '密码修改成功'})
```

### 2. 路由配置

```python
# apps/users/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import UserViewSet

router = DefaultRouter()
router.register(r'users', UserViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

```python
# myproject/urls.py
from django.contrib import admin
from django.urls import path, include
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('apps.users.urls')),
    path('api/', include('apps.blog.urls')),
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

---

## API 开发

### 1. Serializer

```python
# apps/users/serializers.py
from rest_framework import serializers
from django.contrib.auth import get_user_model
from django.contrib.auth.password_validation import validate_password

User = get_user_model()


class UserSerializer(serializers.ModelSerializer):
    """用户序列化器"""
    
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'bio', 'avatar', 'created_at']
        read_only_fields = ['id', 'created_at']


class UserCreateSerializer(serializers.ModelSerializer):
    """用户创建序列化器"""
    password = serializers.CharField(write_only=True, validators=[validate_password])
    password_confirm = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'password_confirm']

    def validate(self, attrs):
        if attrs['password'] != attrs['password_confirm']:
            raise serializers.ValidationError({'password': '两次密码不一致'})
        return attrs

    def create(self, validated_data):
        validated_data.pop('password_confirm')
        user = User.objects.create_user(**validated_data)
        return user
```

### 2. 分页

```python
# core/pagination.py
from rest_framework.pagination import PageNumberPagination


class StandardPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
```

### 3. 过滤

```python
# apps/blog/filters.py
from django_filters import rest_framework as filters
from apps.blog.models import Post


class PostFilter(filters.FilterSet):
    status = filters.CharFilter(field_name='status')
    category = filters.CharFilter(field_name='category__slug')
    author = filters.NumberFilter(field_name='author__id')
    created_after = filters.DateTimeFilter(field_name='created_at', lookup_expr='gte')

    class Meta:
        model = Post
        fields = ['status', 'category', 'author']
```

---

## 安全建议

### 1. 安全设置

```python
# settings.py

# HTTPS
SECURE_SSL_REDIRECT = os.getenv('SECURE_SSL_REDIRECT', 'False') == 'True'
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True

# 内容安全
SECURE_CONTENT_TYPE_NOSNIFF = True

# HSTS
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```

### 2. 密码验证

```python
# settings.py
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
        'OPTIONS': {'min_length': 8},
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
```

### 3. CORS 配置

```python
# settings.py
CORS_ALLOWED_ORIGINS = os.getenv(
    'CORS_ALLOWED_ORIGINS',
    'http://localhost:3000'
).split(',')

CORS_ALLOW_CREDENTIALS = True
```

---

## 部署指南

### 1. Gunicorn

```bash
# gunicorn_config.py
bind = '0.0.0.0:8000'
workers = 4
worker_class = 'sync'
timeout = 120
keepalive = 5

# 日志
accesslog = '-'
errorlog = '-'
loglevel = 'info'
```

```bash
gunicorn -c gunicorn_config.py myproject.wsgi:application
```

### 2. Docker

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

RUN python manage.py collectstatic --noinput

EXPOSE 8000

CMD ["gunicorn", "-c", "gunicorn_config.py", "myproject.wsgi:application"]
```

```yaml
# docker-compose.yml
version: '3.9'

services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DEBUG=False
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db
      - redis

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=mydb
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  postgres_data:
```

### 3. Nginx

```nginx
# /etc/nginx/sites-available/myproject
upstream myproject {
    server 127.0.0.1:8000;
}

server {
    listen 80;
    server_name example.com;

    location /static/ {
        alias /var/www/myproject/static/;
    }

    location /media/ {
        alias /var/www/myproject/media/;
    }

    location / {
        proxy_pass http://myproject;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

## 常用命令

```bash
# 开发
python manage.py runserver
python manage.py runserver 0.0.0.0:8000

# 数据库
python manage.py makemigrations
python manage.py migrate
python manage.py showmigrations

# Django Shell
python manage.py shell

# 测试
python manage.py test
pytest

# 代码检查
python manage.py check
flake8 .
black .

# 管理命令
python manage.py createsuperuser
python manage.py changepassword username
python manage.py flush  # 清空数据库
```

---

## 测试

```python
# apps/users/tests/test_models.py
from django.test import TestCase
from django.contrib.auth import get_user_model

User = get_user_model()


class UserModelTest(TestCase):
    def test_create_user(self):
        user = User.objects.create_user(
            username='testuser',
            email='test@example.com',
            password='testpass123'
        )
        self.assertEqual(user.username, 'testuser')
        self.assertFalse(user.is_staff)
        self.assertTrue(user.check_password('testpass123'))

    def test_create_superuser(self):
        user = User.objects.create_superuser(
            username='admin',
            email='admin@example.com',
            password='adminpass123'
        )
        self.assertTrue(user.is_staff)
        self.assertTrue(user.is_superuser)
```

---

## 性能优化

### 1. 数据库查询优化

```python
# ✅ 正确：使用 select_related
posts = Post.objects.select_related('author', 'category').all()

# ✅ 正确：使用 prefetch_related
users = User.objects.prefetch_related('posts').all()

# ✅ 正确：使用 only 减少查询字段
users = User.objects.only('id', 'username', 'email')

# ✅ 正确：使用 defer 排除不需要的字段
users = User.objects.defer('password', 'last_login')
```

### 2. 缓存

```python
# views.py
from django.core.cache import cache

def get_popular_posts():
    posts = cache.get('popular_posts')
    if not posts:
        posts = Post.objects.filter(status='published')[:10]
        cache.set('popular_posts', posts, 3600)  # 缓存1小时
    return posts
```

### 3. 索引

```python
class Meta:
    indexes = [
        models.Index(fields=['created_at']),
        models.Index(fields=['status', '-created_at']),
        models.Index(fields=['author', 'created_at']),
    ]
```

---

## 常用第三方包

| 类别 | 包 | 说明 |
|------|-----|------|
| API | djangorestframework | REST API 框架 |
| API | django-filter | 过滤 |
| API | djangorestframework-csv | CSV 导出 |
| 管理 | django-cors-headers | CORS 支持 |
| 管理 | django-allauth | 社交登录 |
| 任务 | celery | 异步任务 |
| 测试 | pytest-django | pytest 集成 |
| 部署 | gunicorn | WSGI 服务器 |
| 部署 | whitenoise | 静态文件服务 |

---

## 参考资源

- [Django 官方文档](https://docs.djangoproject.com/)
- [Django REST Framework](https://www.django-rest-framework.org/)
- [Two-Scoops of Django](https://www.feldroy.com/books/two-scoops-of-django-3-edition)
- [Django Blog Tutorial](https://developer.mozilla.org/en-US/docs/Learn/Server-side/Django)

---

## 前端框架集成 (Vite + React/Vue)

### 1. 项目结构

```
myproject/
├── backend/                 # Django 后端
│   ├── myproject/
│   ├── apps/
│   ├── requirements.txt
│   └── manage.py
├── frontend/               # Vite 前端
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── api/
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── public/
│   ├── index.html
│   ├── vite.config.js
│   ├── package.json
│   └── .env
├── docker-compose.yml
├── nginx/
│   └── default.conf
└── README.md
```

### 2. 前端初始化 (Vite + React)

```bash
# 创建前端目录
mkdir frontend && cd frontend

# 初始化 Vite + React
npm create vite@latest . -- --template react

# 安装依赖
npm install

# 安装额外依赖
npm install axios react-router-dom zustand @tanstack/react-query
npm install -D tailwindcss postcss autoprefixer
```

### 3. Vite 配置

```javascript
// vite.config.js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import path from 'path'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
      '/admin': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      },
    },
  },
  build: {
    outDir: '../static/dist',
    emptyOutDir: true,
  },
})
```

### 4. 环境变量

```bash
# frontend/.env
VITE_API_URL=http://localhost:8000
VITE_WS_URL=ws://localhost:8000
```

### 5. API 客户端

```javascript
// src/api/axios.js
import axios from 'axios'

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || '/api',
  withCredentials: true,
})

// 请求拦截器
api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// 响应拦截器
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token')
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)

export default api
```

### 6. React 组件示例

```jsx
// src/pages/Login.jsx
import { useState } from 'react'
import api from '../api/axios'

export default function Login() {
  const [username, setUsername] = useState('')
  const [password, setPassword] = useState('')

  const handleLogin = async (e) => {
    e.preventDefault()
    try {
      const { data } = await api.post('/auth/login/', { username, password })
      localStorage.setItem('token', data.access)
      window.location.href = '/'
    } catch (error) {
      alert('登录失败')
    }
  }

  return (
    <form onSubmit={handleLogin}>
      <input
        type="text"
        value={username}
        onChange={(e) => setUsername(e.target.value)}
        placeholder="用户名"
      />
      <input
        type="password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="密码"
      />
      <button type="submit">登录</button>
    </form>
  )
}
```

---

## Docker Compose 部署

### 1. Docker 配置

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    gcc \
    postgresql-client \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制代码
COPY . .

# 收集静态文件
RUN python manage.py collectstatic --noinput

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["gunicorn", "--config", "gunicorn_config.py", "myproject.wsgi:application"]
```

```dockerfile
# frontend/Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 2. Nginx 配置

```nginx
# nginx/default.conf
upstream backend {
    server backend:8000;
}

server {
    listen 80;
    server_name localhost;

    # 前端静态文件
    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # API 代理
    location /api/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Django Admin
    location /admin/ {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # 静态文件
    location /static/ {
        proxy_pass http://backend;
    }

    location /media/ {
        proxy_pass http://backend;
    }
}
```

### 3. Docker Compose 配置

```yaml
# docker-compose.yml
version: '3.9'

services:
  # PostgreSQL 数据库
  db:
    image: postgres:15-alpine
    container_name: myproject_db
    restart: always
    environment:
      POSTGRES_DB: myproject
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD:-changeme}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis 缓存
  redis:
    image: redis:7-alpine
    container_name: myproject_redis
    restart: always
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Django 后端
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: myproject_backend
    restart: always
    environment:
      - DEBUG=False
      - DATABASE_URL=postgres://postgres:${DB_PASSWORD:-changeme}@db:5432/myproject
      - REDIS_URL=redis://redis:6379/0
      - SECRET_KEY=${SECRET_KEY:-changeme}
      - ALLOWED_HOSTS=${ALLOWED_HOSTS:-localhost}
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./backend:/app
      - static_volume:/app/staticfiles
      - media_volume:/app/media

  # React 前端
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: myproject_frontend
    restart: always
    depends_on:
      - backend

  # Nginx 反向代理
  nginx:
    image: nginx:alpine
    container_name: myproject_nginx
    restart: always
    ports:
      - "${HTTP_PORT:-80}:80"
      - "${HTTPS_PORT:-443}:443"
    volumes:
      - ./nginx:/etc/nginx/conf.d
      - static_volume:/app/staticfiles
      - media_volume:/app/media
    depends_on:
      - backend
      - frontend

volumes:
  postgres_data:
  redis_data:
  static_volume:
  media_volume:
```

### 4. 环境变量文件

```bash
# .env
# Django
DEBUG=False
SECRET_KEY=your-super-secret-key-change-in-production
ALLOWED_HOSTS=localhost,yourdomain.com

# Database
DB_PASSWORD=changeme

# Ports
HTTP_PORT=80
HTTPS_PORT=443
```

### 5. 启动命令

```bash
# 开发环境
docker-compose up -d

# 生产环境
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 查看日志
docker-compose logs -f

# 停止
docker-compose down

# 重启服务
docker-compose restart backend

# 数据库迁移
docker-compose exec backend python manage.py migrate

# 创建超级用户
docker-compose exec backend python manage.py createsuperuser
```

### 6. 生产环境额外配置

```yaml
# docker-compose.prod.yml
services:
  backend:
    environment:
      - DEBUG=False
      - SECURE_SSL_REDIRECT=True
      - SESSION_COOKIE_SECURE=True
      - CSRF_COOKIE_SECURE=True
      - SECURE_HSTS_SECONDS=31536000

  nginx:
    volumes:
      - ./ssl:/etc/nginx/ssl
```

### 7. Makefile 简化操作

```makefile
# Makefile
.PHONY: up down logs migrate superuser build

up:
	docker-compose up -d

down:
	docker-compose down

logs:
	docker-compose logs -f

migrate:
	docker-compose exec backend python manage.py makemigrations
	docker-compose exec backend python manage.py migrate

superuser:
	docker-compose exec backend python manage.py createsuperuser

build:
	docker-compose build

restart:
	docker-compose restart

shell:
	docker-compose exec backend sh
```

---

## 完整项目工作流

### 开发环境

```bash
# 1. 克隆项目
git clone https://github.com/yourusername/myproject.git
cd myproject

# 2. 复制环境变量
cp .env.example .env

# 3. 启动开发环境
docker-compose up -d

# 4. 数据库迁移
docker-compose exec backend python manage.py migrate

# 5. 创建管理员
docker-compose exec backend python manage.py createsuperuser

# 6. 访问
# 前端: http://localhost
# 后端 API: http://localhost/api
# Admin: http://localhost/admin
```

### 生产部署

```bash
# 1. 配置环境变量
export DOMAIN=yourdomain.com
export SECRET_KEY=your-secret-key

# 2. 构建并启动
docker-compose -f docker-compose.yml -f docker-compose.prod.yml up -d --build

# 3. 检查状态
docker-compose ps

# 4. 查看日志
docker-compose logs -f nginx
```

