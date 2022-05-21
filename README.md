## DRF＋CDN版Vue.js＋Cookie認証（Django REST framework）
『現場で使える Django REST Framework の教科書』

### 手順

```
python --version  
→Python 3.10.0  
pip --version  
→pip 22.1 from /home/nisigaki/.pyenv/versions/3.10.0/lib/python3.10/site-packages/pip (python 3.10)  

pip list | grep Django  
→Django 4.0.1  
pip install djangorestframework  
→Successfully installed djangorestframework-3.13.1 pytz-2022.1  

// 最初のコマンド  
django-admin startproject config.  
python manage.py startapp shop  
python manage.py startapp apiv1  

// マイグレーション  
python manage.py makemigrations shop  
python manage.py migrate  

// ユーザー  
python manage.py createsuperuser  
admin / admin@example.com / pass12345  

// 起動
python manage.py runserver  
```

### テーブル一覧
```
auth_group                  book  
auth_group_permissions      django_admin_log  
auth_permission             django_content_type  
auth_user                   django_migrations  
auth_user_groups            django_session
auth_user_user_permissions
```

  

## ソース解説

### アプリが2個
- shop
- apiv1


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


#### ◆router.register('books', views.BookViewSet)

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


#### ◆path('api-auth/', include('rest_framework.urls')),
- api-auth/login/
- api-auth/logout/


### フロントエンド
- templates/index.html に埋め込まれている

　
## 質問

- テーブル一覧で、認証に関わっているのはどれか

- DefaultRouter と SimpleRouter 何が違う？一個ではダメな理由？
```
This router is similar to SimpleRouter as above, but additionally includes a default API root view, that returns a response containing hyperlinks to all the list views. It also generates routes for optional .json style format suffixes.  
https://www.django-rest-framework.org/api-guide/routers/#defaultrouter
```


