## DRF＋CDN版Vue.js＋Cookie認証（Django REST framework）
『現場で使える Django REST Framework の教科書』

### 構成・バージョン

```
// 作業場所
/home/nisigaki/django03-drf/drf-cdn-vue2-sample

// 構成
.
|-config/
| |-settings.py
| |-urls.py
| |-wsgi.py
|-apiv1/
| |-apps.py
| |-migrations/
| |-serializers.py
| |-urls.py
| |-views.py
|-shop/
| |-admin.py
| |-apps.py
| |-migrations/
| |-models.py
|-templates/
| |-index.html
|-manage.py*
|-README.md

python --version
→Python 3.10.0
pip --version
→pip 22.1 from /home/nisigaki/.pyenv/versions/3.10.0/lib/python3.10/site-packages/pip (python 3.10)

pip list | grep -e Django -e django
====
Django                        4.0.1
django-cors-headers           3.12.0
django-extensions             3.1.5
django-templated-mail         1.1.1
django-widget-tweaks          1.4.12
djangorestframework           3.13.1
djangorestframework-simplejwt 4.8.0
social-auth-app-django        4.0.0
```


### 手順（コマンド）

```
// インストール
pip install Django
pip install djangorestframework

// 新規作成
django-admin startproject config.
python manage.py startapp shop
python manage.py startapp apiv1

// マイグレーション
python manage.py makemigrations shop
python manage.py migrate

// ユーザー作成
python manage.py createsuperuser
admin / admin@example.com / pass12345

// 起動
python manage.py runserver
```


### 手順（コーディング）

```
// setting.py 改修

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # 3rd party apps
    'rest_framework',

    # My applications
    'apiv1.apps.Apiv1Config',
    'shop.apps.ShopConfig',
]

LANGUAGE_CODE = 'ja'
TIME_ZONE = 'Asia/Tokyo'

TEMPLATES = [
    {
        "BACKEND": "django.template.backends.django.DjangoTemplates",
        "DIRS": [os.path.join(BASE_DIR, "templates")],　←

# Authentication
LOGIN_REDIRECT_URL = '/'
LOGOUT_REDIRECT_URL = 'rest_framework:login'

# REST Framework
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}

// shop アプリ作成

shop/models.py
====
import uuid
from django.db import models

class Book(models.Model):
    """本モデル"""

    class Meta:
        db_table = 'book'

    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    title = models.CharField(verbose_name='タイトル', max_length=20)
    price = models.IntegerField(verbose_name='価格', null=True, blank=True)
    created_at = models.DateTimeField(verbose_name='登録日時',
                                      auto_now_add=True)

    def __str__(self):
        return self.title
====

shop/admin.py
====
from django.contrib import admin
from .models import Book

class BookModelAdmin(admin.ModelAdmin):
    list_display = ('title', 'price', 'id', 'created_at')
    ordering = ('-created_at',)
    readonly_fields = ('id', 'created_at')

admin.site.register(Book, BookModelAdmin)
====

// apivi アプリ作成

apiv1/serializers.py
====
from rest_framework import serializers
from shop.models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'price']
====

apiv1/views.py
====
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from shop.models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    """BookオブジェクトのCRUDをおこなうAPI"""

    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [IsAuthenticatedOrReadOnly]
====

// ルーティング設定

config/urls.py
====
from django.contrib import admin
from django.urls import path, include
from django.views.generic import TemplateView

urlpatterns = [
    path('admin/', admin.site.urls),
    path('', TemplateView.as_view(template_name='index.html')),
    path('api-auth/', include('rest_framework.urls')),
    path('api/v1/', include('apiv1.urls')),
]
====

apiv1/urls.py
====
from django.urls import path, include
from rest_framework import routers
from . import views

router = routers.DefaultRouter()
router.register('books', views.BookViewSet)

app_name = 'apiv1'
urlpatterns = [
    path('', include(router.urls)),
]
====
```


## テーブル一覧
```
auth_group                  book  
auth_group_permissions      django_admin_log  
auth_permission             django_content_type  
auth_user                   django_migrations  
auth_user_groups            django_session
auth_user_user_permissions
```

　
## ソース解説

#### アプリが2個
- shop
- apiv1

#### フロントエンド
templates/index.html に埋め込まれている

#### ログイン判定
```
templates/index.html
====
{% if user.is_authenticated %}

config/settings.py
====
INSTALLED_APPS = [
    "django.contrib.auth",
```

#### クライアント側で、サーバー側の認証結果を受け取る
```
templates/index.html
====
if (status === 400) {
  // バリデーションNG
  this.message.warnings = [].concat.apply(
    [],
    Object.values(error.response.data)
  );
} else if (status === 401) {
  // 認証エラー
  this.message.error = "認証エラー";
} else if (status === 403) {
  // 権限エラー
  this.message.error = "権限エラーです。";
} else {
  // その他のエラー
  this.message.error = "想定外のエラーです。";
}
```

#### axiosの使い方、特にCSRFトークンの送信設定
```
templates/index.html
====
<!-- axios -->
<script src="https://unpkg.com/axios@0.21.1/dist/axios.min.js"></script>
<script>
  // CSRFトークンの送信設定
  axios.defaults.xsrfCookieName = "csrftoken";
  axios.defaults.xsrfHeaderName = "X-CSRFToken";

  // APIクライアント
  const api = axios.create({
    baseURL: "/api/v1/",
    timeout: 5000,
    headers: {
"Content-Type": "application/json",
"X-Requested-With": "XMLHttpRequest"
    }
  });

templates/index.html
====
submitSave: function() {
  this.clearMessages();
  api({
    // 登録済みかどうかでHTTPメソッドとエンドポイントを切り替える
    method: this.isCreated ? "put" : "post",
    url: this.isCreated
      ? "/books/" + this.form.book.id + "/"
      : "/books/",
    data: {
      id: this.form.book.id,
      title: this.form.book.title,
      price: this.form.book.price
    }
  })
```

#### Stateとその更新。ここではStateを直に書き換えている。
```
templates/index.html
====
data: {
  // 入力フォームの内容
  form: {
    book: {
      title: "",
      price: 0
    }
  },
  // メッセージエリアに表示するメッセージ
  message: {
    info: "",
    warnings: [],
    error: ""
  }
},

templates/index.html
====
.then(response => {
  this.message.info = this.isCreated
    ? "更新しました。"
    : "登録しました。";
  this.form.book = response.data;
})
```

#### Stateの参照（message）
```
templates/index.html
====
<!-- メッセージエリア -->
<div id="messages">
  <b-alert variant="danger" show v-show="message.error" class="mb-0">
    ${ message.error }
  </b-alert>

Stateの参照（form）

templates/index.html
====
<!-- メインエリア -->
<div id="main-page">
  <main class="container mt-5 p-5">
    <p class="h5 mb-5">ホーム</p>
    <b-form v-on:submit.prevent="submitSave">
      <div class="row form-group">
        <label class="col-sm-3 col-form-label">タイトル</label>
        <div class="col-sm-8">
          <b-input type="text" class="form-control" v-model="form.book.title" />
        </div>
      </div>
```
        



### ルーティング

- config/urls.py 
```
urlpatterns = [
    path('admin/', admin.site.urls),
    path('', TemplateView.as_view(template_name='index.html')),
    path('api-auth/', include('rest_framework.urls')),
    path('api/v1/', include('apiv1.urls')),
]
```
- shop
```
なし
```
- apiv1/urls.py
```
router = routers.DefaultRouter()
router.register('books', views.BookViewSet)
urlpatterns = [
    path('', include(router.urls)),
]
```


#### router.register('books', views.BookViewSet)

"http://127.0.0.1:8000/api/v1/books/" が下記を返す
```
GET /api/v1/books/
HTTP 200 OK
Allow: GET, POST, HEAD, OPTIONS
Content-Type: application/json
Vary: Accept

[
    {
        "id": "808df84a-087c-40d1-8670-4f37daa7c957",
        "title": "ほん１",
        "price": 1000
    }
]
```


#### path('api-auth/', include('rest_framework.urls')),
- api-auth/login/
- api-auth/logout/

　
## 質問

- テーブル一覧で、認証に関わっているのはどれか

- DefaultRouter と SimpleRouter 何が違う？一個ではダメな理由？
```
This router is similar to SimpleRouter as above, but additionally includes a default API root view, that returns a response containing hyperlinks to all the list views. It also generates routes for optional .json style format suffixes.  
https://www.django-rest-framework.org/api-guide/routers/#defaultrouter
```

- ここ、入れ子の時はどう書くの？（Book-Author みたいなリレーション）
```
router = routers.DefaultRouter()
router.register('books', views.BookViewSet)
```


