# Django 开发指南

Django 是一个高级 Python Web 框架，鼓励快速开发和简洁实用的设计。

## 目录

1. [项目结构](#项目结构)
2. [环境配置](#环境配置)
3. [最佳实践](#最佳实践)
4. [模型设计](#模型设计)
5. [视图与路由](#视图与路由)
6. [服务层](#服务层)
7. [认证与权限](#认证与权限)
8. [缓存与性能](#缓存与性能)
9. [异步任务](#异步任务)
10. [管理命令](#管理命令)
11. [安全建议](#安全建议)
12. [部署指南](#部署指南)

---

## 项目结构

推荐的单应用模块化结构（参考 meta-space）：

```
myproject/
├── myproject/                 # 项目配置
│   ├── __init__.py
│   ├── settings.py           # 项目设置
│   ├── urls.py               # 根路由
│   ├── wsgi.py               # WSGI 入口
│   └── asgi.py               # ASGI 入口
├── app/                      # 主应用（推荐单应用）
│   ├── __init__.py
│   ├── apps.py
│   ├── models.py             # 所有模型
│   ├── views.py              # API 视图
│   ├── views_auth.py         # 认证视图
│   ├── views_dashboard.py    # 仪表盘视图
│   ├── serializers.py        # 序列化器
│   ├── urls.py               # 路由配置
│   ├── tasks.py              # Celery 任务
│   ├── services/             # 服务层 ⭐
│   │   ├── __init__.py
│   │   ├── base.py           # 基类
│   │   ├── symbols_service.py
│   │   └── daily_kline_service.py
│   ├── management/           # Django 管理命令
│   │   └── commands/
│   │       ├── __init__.py
│   │       └── populate_xxx.py
│   ├── migrations/
│   ├── utils/
│   │   ├── __init__.py
│   │   └── helpers.py
│   └── admin.py
├── static/                   # 静态文件
├── media/                    # 用户上传
├── requirements.txt
├── .env
├── manage.py
├── docker-compose.yml
└── Dockerfile
```

**特点：**
- 单应用结构，避免循环导入
- services/ 目录封装外部 API 调用
- management/commands/ 放置数据同步命令
- tasks.py 集中管理异步任务

---

## 环境配置

### 1. 依赖管理

```bash
# requirements.txt
Django>=4.2,<5.0
djangorestframework>=3.14,<4.0
djangorestframework-simplejwt>=5.3
django-cors-headers>=4.3
django-redis>=5.4
django-celery-beat>=2.5
celery>=5.3
redis>=5.0
mysqlclient>=2.2
python-dotenv>=1.0
gunicorn>=21.0
```

### 2. 环境变量 (.env)

```bash
# .env
DEBUG=False
SECRET_KEY=your-secret-key-change-in-production
ALLOWED_HOSTS=localhost,127.0.0.1,yourdomain.com

# 数据库 (MySQL)
DB_NAME=myproject
DB_USER=myproject
DB_PASSWORD=your_password
DB_HOST=localhost
DB_PORT=3306

# Redis
REDIS_URL=redis://localhost:6379/0

# Celery
CELERY_BROKER_URL=redis://localhost:6379/0
CELERY_RESULT_BACKEND=redis://localhost:6379/0

# JWT
JWT_SECRET_KEY=your_jwt_secret
```

### 3. settings.py 配置

```python
# settings.py
import os
from pathlib import Path
from datetime import timedelta

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = os.getenv('SECRET_KEY')
DEBUG = os.getenv('DEBUG', 'False').lower() == 'true'
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', '*').split(',')

# 应用
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'corsheaders',
    'rest_framework',
    'django_celery_beat',
    'app',
]

# 中间件
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'corsheaders.middleware.CorsMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# 数据库 (MySQL)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': os.getenv('DB_NAME'),
        'USER': os.getenv('DB_USER'),
        'PASSWORD': os.getenv('DB_PASSWORD'),
        'HOST': os.getenv('DB_HOST', 'localhost'),
        'PORT': os.getenv('DB_PORT', '3306'),
        'OPTIONS': {
            'charset': 'utf8mb4',
        },
    }
}

# REST Framework
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
    'DEFAULT_PERMISSION_CLASSES': (
        'rest_framework.permissions.IsAuthenticated',
    ),
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '500/day',
        'user': '5000/day',
    },
}

# JWT 配置
SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(hours=2),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=7),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'AUTH_HEADER_TYPES': ('Bearer',),
}

# Redis 缓存
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': os.getenv('REDIS_URL', 'redis://localhost:6379/1'),
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'myproject',
        'TIMEOUT': 300,
    }
}

# Celery 配置
CELERY_BROKER_URL = os.getenv('CELERY_BROKER_URL')
CELERY_RESULT_BACKEND = os.getenv('CELERY_RESULT_BACKEND')
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Asia/Shanghai'

# Celery Beat 周期任务
from celery.schedules import crontab
CELERY_BEAT_SCHEDULE = {
    'daily-task': {
        'task': 'app.tasks.daily_task',
        'schedule': crontab(hour=2, minute=0),
    },
}

# 自定义用户模型
AUTH_USER_MODEL = 'app.User'
```

---

## 最佳实践

### 1. 项目初始化

```bash
# 创建项目
django-admin startproject myproject .

# 创建应用
python manage.py startapp app

# 迁移
python manage.py makemigrations
python manage.py migrate

# 超级用户
python manage.py createsuperuser
```

### 2. 代码规范

```python
# ✅ 正确：使用路径导入
from app.models import User
from app.services import StockService

# ✅ 正确：类型注解
from typing import Optional, List
from django.db.models import QuerySet

def get_user(pk: int) -> Optional[User]:
    return User.objects.filter(pk=pk).first()

# ✅ 正确：使用视图集
from rest_framework import viewsets

class StockViewSet(viewsets.ViewSet):
    pass
```

---

## 模型设计

### 1. 基础模型

```python
# app/models.py
from django.db import models
from django.contrib.auth.models import AbstractUser
from django.utils import timezone


class User(AbstractUser):
    """扩展用户模型"""
    phone = models.CharField(max_length=20, blank=True, verbose_name='手机号')
    avatar = models.URLField(blank=True, verbose_name='头像')
    api_calls_today = models.IntegerField(default=0, verbose_name='今日API调用次数')
    api_calls_reset_at = models.DateTimeField(default=timezone.now, verbose_name='API调用重置时间')
    is_premium = models.BooleanField(default=False, verbose_name='是否Premium用户')
    created_at = models.DateTimeField(auto_now_add=True, verbose_name='注册时间', db_index=True)
    updated_at = models.DateTimeField(auto_now=True, verbose_name='更新时间')

    class Meta:
        db_table = 'user'
        verbose_name = '用户'
        verbose_name_plural = '用户'

    def __str__(self):
        return self.username


class TimeStampedModel(models.Model):
    """时间戳基类"""
    created_at = models.DateTimeField(auto_now_add=True, verbose_name='创建时间')
    updated_at = models.DateTimeField(auto_now=True, verbose_name='更新时间')

    class Meta:
        abstract = True


class UUIDModel(models.Model):
    """UUID主键基类"""
    id = models.UUIDField(primary_key=True, default=models.UUIDField(default=uuid.uuid4, editable=False))

    class Meta:
        abstract = True
```

### 2. 关系模型

```python
class UserFavorite(TimeStampedModel):
    """用户收藏"""
    user = models.ForeignKey(
        User, 
        on_delete=models.CASCADE, 
        related_name='favorites',
        verbose_name='用户'
    )
    symbol = models.CharField(max_length=20, verbose_name='股票代码')
    market = models.CharField(max_length=10, verbose_name='市场')

    class Meta:
        db_table = 'user_favorite'
        verbose_name = '用户收藏'
        verbose_name_plural = '用户收藏'
        unique_together = [['user', 'symbol', 'market']]
        indexes = [
            models.Index(fields=['user', 'market'], name='user_fav_user_market_idx')
        ]

    def __str__(self):
        return f"{self.user.username} - {self.symbol}"
```

### 3. 数据模型示例

```python
class StockList(TimeStampedModel):
    """股票列表"""
    symbol = models.CharField(max_length=20, unique=True, verbose_name='股票代码')
    company_name = models.CharField(max_length=100, verbose_name='公司名称')
    exchange = models.CharField(max_length=10, verbose_name='交易所', db_index=True)
    
    # A股字段
    ts_code = models.CharField(max_length=20, verbose_name='TS代码', blank=True)
    area = models.CharField(max_length=50, verbose_name='地域', blank=True)
    industry = models.CharField(max_length=50, verbose_name='行业', blank=True)
    list_date = models.DateField(verbose_name='上市日期', null=True, blank=True, db_index=True)
    
    # 美股字段
    sector = models.CharField(max_length=50, verbose_name='行业板块', blank=True)
    market_cap = models.BigIntegerField(verbose_name='市值', null=True, blank=True, db_index=True)
    price = models.DecimalField(max_digits=20, decimal_places=2, verbose_name='当前价格', null=True, blank=True)
    
    is_actively_trading = models.BooleanField(default=True, verbose_name='是否交易中')
    data_source = models.CharField(max_length=20, verbose_name='数据源', blank=True, db_index=True)

    class Meta:
        db_table = 'stock_list'
        verbose_name = '股票列表'
        verbose_name_plural = '股票列表'
        indexes = [
            models.Index(fields=['exchange', 'is_actively_trading']),
        ]

    def __str__(self):
        return f"{self.symbol} - {self.company_name}"
```

### 4. 字段命名规范

| 字段类型 | 命名示例 | 说明 |
|---------|---------|------|
| 外键 | `user`, `category` | 使用模型名小写 |
| 布尔 | `is_active`, `is_premium` | is_ 前缀 |
| 时间 | `created_at`, `updated_at` | auto_now_add/auto_now |
| 索引 | `db_index=True` | 频繁查询字段加索引 |
| 关联 | `related_name` | 便于反向查询 |

---

## 视图与路由

### 1. API 视图

```python
# app/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework.throttling import AnonRateThrottle, UserRateThrottle
from django.core.cache import cache
from django.db.models import Q

class StockListView(APIView):
    """股票列表视图"""
    throttle_classes = [AnonRateThrottle, UserRateThrottle]
    
    def get(self, request):
        # 缓存查询
        cache_key = 'stock_list'
        data = cache.get(cache_key)
        
        if data is None:
            from app.models import StockList
            stocks = StockList.objects.filter(is_actively_trading=True)
            data = list(stocks.values())
            cache.set(cache_key, data, 300)  # 5分钟缓存
        
        return Response(data)
```

### 2. 路由配置

```python
# app/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import StockListView
from .views_auth import LoginView, LogoutView
from .views_dashboard import DashboardView

router = DefaultRouter()

urlpatterns = [
    path('', include(router.urls)),
    path('stocks/', StockListView.as_view(), name='stock-list'),
    path('auth/login/', LoginView.as_view(), name='login'),
    path('auth/logout/', LogoutView.as_view(), name='logout'),
    path('dashboard/', DashboardView.as_view(), name='dashboard'),
]
```

```python
# myproject/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('app.urls')),
]
```

### 3. 序列化器

```python
# app/serializers.py
from rest_framework import serializers
from django.contrib.auth import get_user_model
from django.contrib.auth.password_validation import validate_password

User = get_user_model()


class UserSerializer(serializers.ModelSerializer):
    class Meta:
        model = User
        fields = ['id', 'username', 'email', 'phone', 'avatar', 'is_premium', 'created_at']
        read_only_fields = ['id', 'created_at']


class UserCreateSerializer(serializers.ModelSerializer):
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

---

## 服务层

### 1. 服务基类

```python
# app/services/base.py
from abc import ABC, abstractmethod
from typing import List, Dict, Any, Optional
import logging

logger = logging.getLogger(__name__)


class BaseService(ABC):
    """服务基类"""
    
    def __init__(self):
        self.logger = logger
    
    @abstractmethod
    def fetch_symbols(self, **kwargs) -> List[Dict]:
        """获取标的列表"""
        pass
    
    @abstractmethod
    def fetch_daily_kline(self, symbol: str, **kwargs) -> List[Dict]:
        """获取日K线"""
        pass
    
    def log_error(self, message: str, **kwargs):
        self.logger.error(message, extra=kwargs)
    
    def log_info(self, message: str, **kwargs):
        self.logger.info(message, extra=kwargs)
```

### 2. 数据服务示例

```python
# app/services/tushare_service.py
import tushare as ts
from typing import List, Dict
from .base import BaseService


class TushareService(BaseService):
    """Tushare 数据服务"""
    
    def __init__(self, token: str = None):
        super().__init__()
        self.pro = ts.pro_api(token)
    
    def fetch_symbols(self, market: str = 'cn', symbol_type: str = 'stock') -> List[Dict]:
        """获取股票列表"""
        try:
            df = self.pro.stock_basic(
                exchange='',
                list_status='L',
                fields='ts_code,symbol,name,area,industry,list_date'
            )
            return df.to_dict('records')
        except Exception as e:
            self.log_error('fetch_symbols failed', error=str(e))
            return []
    
    def fetch_daily_kline(self, ts_code: str, start_date: str, end_date: str) -> List[Dict]:
        """获取日K线"""
        try:
            df = self.pro.daily(ts_code=ts_code, start_date=start_date, end_date=end_date)
            return df.to_dict('records')
        except Exception as e:
            self.log_error('fetch_daily_kline failed', error=str(e), ts_code=ts_code)
            return []
```

### 3. 服务注册

```python
# app/services/__init__.py
from .tushare_service import TushareService
from .fmp_service import FMPService

__all__ = ['TushareService', 'FMPService']
```

---

## 认证与权限

### 1. JWT 认证

```python
# settings.py 添加
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ),
}
```

### 2. 自定义权限

```python
# app/permissions.py
from rest_framework import permissions


class IsPremiumUser(permissions.BasePermission):
    def has_permission(self, request, view):
        return request.user and request.user.is_authenticated and request.user.is_premium


class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.user == request.user
```

### 3. 认证视图

```python
# app/views_auth.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from rest_framework_simplejwt.tokens import RefreshToken
from django.contrib.auth import authenticate


class LoginView(APIView):
    def post(self, request):
        username = request.data.get('username')
        password = request.data.get('password')
        
        user = authenticate(username=username, password=password)
        
        if user:
            refresh = RefreshToken.for_user(user)
            return Response({
                'access': str(refresh.access_token),
                'refresh': str(refresh),
                'user': {
                    'id': user.id,
                    'username': user.username,
                    'is_premium': user.is_premium,
                }
            })
        
        return Response(
            {'error': '用户名或密码错误'},
            status=status.HTTP_401_UNAUTHORIZED
        )


class LogoutView(APIView):
    def post(self, request):
        try:
            refresh_token = request.data.get('refresh')
            token = RefreshToken(refresh_token)
            token.blacklist()
            return Response({'message': '登出成功'})
        except Exception:
            return Response({'error': '无效的token'}, status=status.HTTP_400_BAD_REQUEST)
```

---

## 缓存与性能

### 1. Redis 缓存

```python
# 缓存使用
from django.core.cache import cache

# 设置缓存
cache.set('key', 'value', 300)  # 5分钟

# 获取缓存
value = cache.get('key')

# 删除缓存
cache.delete('key')

# 模式删除
cache.delete_pattern('stock_*')
```

### 2. 查询优化

```python
# ✅ 正确：使用 select_related
stocks = StockList.objects.select_related('category').all()

# ✅ 正确：使用 prefetch_related
user = User.objects.prefetch_related('favorites').first()

# ✅ 正确：使用 only/defer
stocks = StockList.objects.only('symbol', 'company_name').all()

# ✅ 正确：使用 values()
data = StockList.objects.values('symbol', 'price')
```

### 3. 索引配置

```python
class Meta:
    indexes = [
        models.Index(fields=['created_at']),
        models.Index(fields=['exchange', 'is_actively_trading']),
        models.Index(fields=['user', '-created_at']),
    ]
```

---

## 异步任务

### 1. Celery 任务

```python
# app/tasks.py
from celery import shared_task
from django.utils import timezone
from datetime import timedelta


@shared_task
def daily_task():
    """每日定时任务"""
    from app.services import StockService
    
    service = StockService()
    symbols = service.fetch_symbols()
    
    # 处理数据
    for symbol in symbols:
        process_symbol.delay(symbol['code'])
    
    return f'Processed {len(symbols)} symbols'


@shared_task
def process_symbol(symbol_code: str):
    """处理单个标的"""
    from app.models import StockList
    
    stock = StockList.objects.filter(symbol=symbol_code).first()
    if stock:
        # 更新逻辑
        pass
    return f'Processed {symbol_code}'


@shared_task
def reset_api_calls():
    """重置每日API调用次数"""
    from app.models import User
    
    User.objects.all().update(
        api_calls_today=0,
        api_calls_reset_at=timezone.now() + timedelta(days=1)
    )
```

### 2. Celery Beat 配置

```python
# settings.py
CELERY_BEAT_SCHEDULE = {
    'daily-task': {
        'task': 'app.tasks.daily_task',
        'schedule': crontab(hour=2, minute=0),  # 每天 02:00
    },
    'reset-api-calls': {
        'task': 'app.tasks.reset_api_calls',
        'schedule': crontab(hour=0, minute=0),  # 每天 00:00
    },
}
```

### 3. 启动 Celery

```bash
# Worker
celery -A myproject worker -l info

# Beat
celery -A myproject beat -l info

# Docker Compose
docker compose up -d celery_worker celery_beat
```

---

## 管理命令

### 1. 数据同步命令

```python
# app/management/commands/populate_stocks.py
from django.core.management.base import BaseCommand
from app.services import TushareService
from app.models import StockList


class Command(BaseCommand):
    help = '同步股票列表'
    
    def add_arguments(self, parser):
        parser.add_argument('--market', type=str, default='cn', help='市场: cn, us, hk')
        parser.add_argument('--batch-size', type=int, default=100, help='批量大小')
    
    def handle(self, *args, **options):
        market = options['market']
        batch_size = options['batch_size']
        
        self.stdout.write(f'Starting sync for market: {market}')
        
        service = TushareService()
        symbols = service.fetch_symbols(market=market)
        
        stocks_to_create = []
        for symbol in symbols:
            stocks_to_create.append(StockList(
                symbol=symbol['symbol'],
                company_name=symbol['name'],
                exchange=symbol.get('exchange', ''),
                data_source=market,
            ))
        
        # 批量创建
        StockList.objects.bulk_create(stocks_to_create, batch_size=batch_size, ignore_conflicts=True)
        
        self.stdout.write(self.style.SUCCESS(f'Synced {len(stocks_to_create)} stocks'))
```

### 2. 断点续传命令

```python
# app/management/commands/populate_daily_kline.py
from django.core.management.base import BaseCommand
from django.utils import timezone
import json
import os


class Command(BaseCommand):
    help = '同步日K线数据（支持断点续传）'
    
    def add_arguments(self, parser):
        parser.add_argument('--market', type=str, required=True, help='市场')
        parser.add_argument('--init', action='store_true', help='全量初始化')
        parser.add_argument('--no-resume', action='store_true', help='忽略断点')
    
    def handle(self, *args, **options):
        market = options['market']
        checkpoint_file = f'/tmp/daily_kline_{market}.json'
        
        # 检查断点
        start_index = 0
        if not options['no_resume'] and os.path.exists(checkpoint_file):
            with open(checkpoint_file, 'r') as f:
                checkpoint = json.load(f)
                start_index = checkpoint.get('index', 0)
                self.stdout.write(f'Resuming from index: {start_index}')
        
        # 执行同步
        total = sync_daily_kline(market, start_index)
        
        # 删除断点文件（完成时）
        if os.path.exists(checkpoint_file):
            os.remove(checkpoint_file)
        
        self.stdout.write(self.style.SUCCESS(f'Synced {total} records'))
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
AUTH_PASSWORD_VALIDATORS = [
    {'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator'},
    {'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator'},
    {'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator'},
    {'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator'},
]
```

### 3. CORS 配置

```python
CORS_ALLOW_ALL_ORIGINS = os.getenv('DEBUG', 'False').lower() == 'true'

# 或指定域名
CORS_ALLOWED_ORIGINS = [
    'http://localhost:3000',
    'http://localhost:5179',
]
```

---

## 部署指南

### 1. Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  db:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping"]

  redis:
    image: redis:7-alpine
    restart: always

  web:
    build: ./backend
    restart: always
    ports:
      - "8000:8000"
    environment:
      - DEBUG=False
      - DB_HOST=db
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      db:
        condition: service_healthy

  celery_worker:
    build: ./backend
    command: celery -A myproject worker -l info
    restart: always
    environment:
      - DB_HOST=db
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis

  celery_beat:
    build: ./backend
    command: celery -A myproject beat -l info
    restart: always
    environment:
      - DB_HOST=db
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - redis

volumes:
  mysql_data:
```

### 2. Dockerfile

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    gcc \
    default-libmysqlclient-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "myproject.wsgi:application"]
```

### 3. Gunicorn 配置

```python
# gunicorn_config.py
bind = '0.0.0.0:8000'
workers = 4
worker_class = 'sync'
timeout = 120
keepalive = 5
loglevel = 'info'
```

### 4. Nginx 配置

```nginx
upstream myproject {
    server web:8000;
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

# 测试
python manage.py test
pytest

# 代码检查
python manage.py check
flake8 .
black .

# 管理
python manage.py createsuperuser
python manage.py shell
```

---

## 常用第三方包

| 类别 | 包 | 说明 |
|------|-----|------|
| API | djangorestframework | REST API 框架 |
| API | djangorestframework-simplejwt | JWT 认证 |
| API | django-cors-headers | CORS 支持 |
| 缓存 | django-redis | Redis 缓存 |
| 任务 | celery | 异步任务 |
| 任务 | django-celery-beat | Celery 定时任务 |
| 数据库 | mysqlclient | MySQL 驱动 |
| 测试 | pytest-django | pytest 集成 |
| 部署 | gunicorn | WSGI 服务器 |

---

## 参考资源

- [Django 官方文档](https://docs.djangoproject.com/)
- [Django REST Framework](https://www.django-rest-framework.org/)
- [Simple JWT](https://django-rest-framework-simplejwt.readthedocs.io/)
- [Celery 文档](https://docs.celeryproject.org/)
