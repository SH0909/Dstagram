# Dstagram 만들기
* 필요한 기능 ```사진 목록, 사진 추가, 업데이트,사진 상세 정보 확인, 사진 삭제, 로그인,로그아웃, 회원가입```

# 프로젝트 만들기
1. python -m venv venv
2. 가상환경 실행:   
source venv/Scripts/activate
3. 장고 설치:   
pip install django
4. 장고 프로젝트 생성:    
django-admin startproject config .
5. 데이터베이스 초기화:   
python manage.py migrate
6. 관리자 계정 생성:   
python manage.py createsuperuser

# Photo앱 만들기-사진 관리
## 1.앱 만들기
* python manage.py startapp photo   
* config/settings.py (photo 앱 등록)
```
INSTALLED_APPS=[
    'photo', //추가
]
```
## 2.모델 만들기
* photo/models.py
```
from django.db import models
from django.contrib.auth.models import User

class Photo(models.Model): 
    author=models.ForeignKey(User,on_delete=models.CASCADE,related_name='user_photos')
    photo=models.ImageField(upload_to='photos/%Y/%m/%d',default='photos/no_image.png')
    text=models.TextField()
    created=models.DateTimeField(auto_now_add=True)
    updated=models.DateTimeField(auto_now=True)
```
1. 모델은 항상 models.Model 상속
2. User 모델은 장고에서 기본적으로 사용하는 사용자 모델이다
3. author :ForeignKey를 사용해서 User테이블과 관계를 만든다   
```on_delete:```연결된 모델이 삭제될때 현재 모델의 값은 어떻게 할건지 정함(CASCADE:연결된 객체가 지워지면 하위 객체도 삭제/PROTECT:하위 객체가 남아있다면 연결된 객체가 지워지지 않음 등등..)   
```related_name:```연결된 객체에서 하위 객체의 목록을 부를때 사용할 이름   
ex)어떤 유저가 작성한 글을 불러 올때는 유저 객체에 user_photo 속성을 참조
4. photo :사진 필드,   
```upload_to:```사진이 업로드 될 경로 /사진 업로드x->default값으로 대체 
ex)photos/년/월/일 폴더에 저장
5. text :사진에 대한 설명을 저장할 텍스트 필드
6. created :글 작성 일을 저장하기 위한 날짜시간 필드 ```auto_now_add:```객체가 추가될때 자동으로 값을 설정
7. updated :글 수정 일을 저장하기 위한 날짜시간 필드 ```auto_now:```객체가 수정될때 자동으로 값을 설정
* Meta 클래스 추가
```
class Photo(models.Model):
    class Meta:
        ordering=['updated']
```
```ordering:```해당 모델의 객체들을 어떤 기준으로 정렬할 것인지 설정하는 옵션   
-updated :업데이트 날짜 내림차순 정렬(앞에 -때면 오름차순 정렬)
* __str__ 메소드 추가
```
class Photo(models.Model):
    def__str__(self):
        return self.author.username+" "+self.created.strftime("%Y-%m-%d %H:%M:%S)
```
작성자의 이름과 글 작성일을 합친 문자열 반환
* get_absolute_url 메소드 추가 (객체의 상세페이지 주소 반환)
```
from django.urls import reverse
class Photo(models.Model):
    def get_absolute_url(self):
        return reverse('photo:photo_detail',args=[str(self.id)])
```
```reverse:```url패턴 이름을 가지고 해당 패턴을 찾아 주소를 만들어주는 함수
* 모델을 만들었으니 데이터 베이스에 적용 해준다   
python manage.py makemigration photo   
```위 코드 실행하면 오류 발생/ ImageField 사용하려면 Pillow모듈 필요->pip install pillow```
* 변경사항 데이터베이스에 적용
python manage.py migrate photo

## 3.관리자 사이트에 모델 등록
* photo/admin.py
```
from django.contrib import admin
from .models import Photo

admin.site.register(Photo)
```
Photo모델 등록
## 4.업로드 폴더 관리
* 앱이 여러개일 경우 여러 앱이 각각의 폴더를 만들어 사진을 업로드하면 프로젝트 루트에 많은 폴더 생성   
->파일들이 모이는 폴더 만들어 관리
* config/settings.py
```
MEDIA_URL='/media/'
MEDIA_ROOT=os.path.join(BASE_DIR,'media')
```
1. MEDIA_ROOT값 설정->어떤 앱에서 업로드를 해도 media 폴더 밑에 각 앱 별로 폴더를 만들고 파일 업로드
2. MEDIA_URL :파일을 브라우저로 서빙할 때 보여줄 가상의 URL

## 5.관리자 페이지 커스터마이징
* photo/admin.py (관리자 사이트에서 보이는 목록 화면 커스터마이징)
```
class PhotoAdmin(admin.ModelAdmin):
    list_display=['id','author','created','updated']
    raw_id_fields=['author']
    list_filter=['created','updated','author']
    search_fields=['text','created']
    ordering=['updated','-created']

admin.site.register(Photo,PhotoAdmin)
```
1. list_display :목록에 보일 필드 설정
2. raw_id_fields :ForeignKey는 연결된 모델의 객체 목록을 출력하고 선택해야 하는데 목록이 길 경우 불편함->값을 써넣는 형태로 바뀌고 검색 기능을 사용해 선택 가능
3. list_filter :필터 기능을 사용해 필드 선택
4. search_fields :검색 기능을 통해 검색할 필드 선택 (ForeignKey 필드는 설정 불가)
5. ordering :관리자 사이트에서 사용할 정렬값 선택

## 6.뷰 만들기
* photo/views.py (목록)
```
from django.shortcuts import render
from .models import Photo

def photo_list(request):
    photos=Photo.objects.all()
    return render(request,'photo/list.html',{'photos':photos})
```
1. 함수형 뷰는 기본 매개변수로 request 설정
2. 사진 객체를 불러오기 위해 Photo모델의 기본 매니저인 object를 이용해 all 메소드 호출 -> 데이터베이스에 저장된 모든 사진 불러옴
3. list.html 렌더링 ,변수 photos전달
* 사진 업로드 뷰
```
class PhotoUploadView(LoginRequiredMixin,CreateView):
    model=Photo
    fields=['photo','text']
    template_name='photo/upload.html'
    def form_valid(self,form):
        form.instance.author_id=self.request.user.id
        if form.is_valid():
            form.instance.save()
            return redirect('/')
        else:
            return self.render_to_response({'form':form})
```
1. template_name에 실제 사용할 템플릿 설정 (photo 폴더에 있는 upload.html 사용)
2. form_valid 메소드 :업로드를 끝낸후 이동할 페이지를 호출하기 위해 사용하는 메소드 ```(오버라이딩 해서 작성자 설정 기능 추가함)```   
작성자=현재 로그인 한 사용자로 설정   
유효하다면 데이터베이스에 저장하고 메인 페이지로 이동 / 이상 있으면 작성 내용 그대로 페이지에 표시    

* 삭제/업데이트
```
class PhotoDeleteView(LoginRequiredMixin,DeleteView):
    model=Photo
    success_url='/'
    template_name='photo/delete.html'

class PhotoUpdateView(LoginRequiredMixin,UpdateView):
    model=Photo
    fields=['photo','text']
    template_name='photo/update.html'
```
success_url은 삭제 성공 후 보여줄 페이지 지정 "/" 메인 페이지로 이동

```제너릭 뷰는 views.py에 작성하는 방법 말고 urls.py에서 작성할 수도 있다(detail뷰는 urls.py에 있음)```
## 7.URL 연결하기
* photo/urls.py
```
from django.urls import path
from django.views.generic.detail import DetailView
from .views import *
from .models import Photo

app_name='photo'

urlpatterns=[
    path('',photo_list,name='photo_list'),
    path('detail/<int:pk>/',DetailView.as_view(model=Photo,template_name='photo/detail.html'),name='photo_detail'),
    path('upload/',PhotoUploadView.as_view(),name='photo_upload'),
    path('delete/<int:pk>/',PhotoDeleteView.as_view(),name='photo_delete'),
    path('update/<int:pk>/',PhotoUpdateView.as_view(),name='photo_update'),
]
```
1. ```app_name:```네임스페이스로 사용되는 값 / 템플릿에서 url 템플릿 태그를 사용할때 app_name이 설정되어 있다면 [app_name:url패턴이름] 형태로 사용
2. 디테일뷰 urls.py에서 인라인 코드로 작성하기   
```as_view 안에 클래스 변수들 설정해 사용```
* config/urls.py (루트에 앱의 urls.py연결)
```
from django.contrib import admin
from django.urls import path,include 

urlpatterns = [
    path('admin/',admin.site.urls),
    path('',include('photo.urls')),
]
```
## 8.템플릿 분리와 확장
* templates/base.html (루트에 templates 폴더 만들고 base.html 파일 만들기)
```
<!DOCTYPE html>
<html lang="en" style="height: 100%;">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link rel="stylesheet" href="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/css/bootstrap.min.css" integrity="sha384-Vkoo8x4CGsO3+Hhxv8T/Q5PaXtkKtu6ug5TOeNV6gBiFeWPGFN9MuhOf23Q9Ifjh" crossorigin="anonymous">
    <script src="https://code.jquery.com/jquery-3.4.1.slim.min.js" integrity="sha384-J6qa4849blE2+poT4WnyKhv5vZF5SrPo0iEjwBvKU7imGFAV0wwj1yYfoRSJoZ+n" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/popper.js@1.16.0/dist/umd/popper.min.js" integrity="sha384-Q6E9RHvbIyZFJoft+2mJbHaEWldlvI9IOYy5n3zV9zzTtmI3UksdQRVvoxMfooAo" crossorigin="anonymous"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/js/bootstrap.min.js" integrity="sha384-wfSDF2E50Y2D1uUdj0O3uMBJnjuUD4Ih7YwaYd1iqfktj0Uod8GCExl3Og8ifwB6" crossorigin="anonymous"></script>
    <title>Dstagram{% block title %}{% endblock %}</title>
</head>
<body style="height:100%;">
    <div class="container" style="min-height: 100%; position: relative; padding-bottom: 50px;">
        <header class="header clearfix">
            <nav class="navbar navbar-expand-lg navbar-light bg-light">
                <a class="navbar-brand" href="/">Dstagram</a>
                <ul class="nav">
                    <li class="nav-item"><a href="/" class="active nav-link">Home</a></li>
                    {% if user.is_authenticated %}
                    <li class="nav-item"><a href="#" class="nav-link">Welcome, {{user.get_username}}</a></li>
                    <li class="nav-item"><a href="{% url 'photo:photo_upload' %}" class="nav-link">Upload</a></li>
                    <li class="nav-item"><a href="{% url 'logout' %}" class="nav-link">Logout</a></li>
                    {% else %}
                    <li class="nav-item"><a href="{% url 'login' %}" class="nav-link">Login</a></li>
                    <li class="nav-item"><a href="{% url 'register' %}" class="nav-link">Signup</a></li>
                    {% endif %}
                </ul>
            </nav>
        </header>
        {% block content %}
        {% endblock %}
        <footer class="footer" style="background-color:rgba(52, 216, 235,0.7); color:white; padding:10px; position:absolute; width:100%;bottom: 0px; left:0px;">
            <p>&copy 2020 SH0909. Powered by Django</p>
        </footer>
    </div>
    
</body>
</html>
```
1. head 안에 있는 css와 js는 부트스트랩이다
2. Dstagram{block title}로 이름 만들어짐
3. user.is_authenticated :로그인 여부 판단 (장고 템플릿에서는 user객체로 유저 정보를 얻어올수 있다..)
4. 로그인 했을때랑 안했을때 보이는 메뉴가 다름 (if 안에 있는건 로그인 했을때 else 안에 있는건 로그인 안했을때)
5. 책에 있는대로 했더니 height가 창보다 작은 경우는 footer가 화면 바닥에 안붙어있고 div 끝에 붙어있는게 맘에 안들었음
```footer 페이지 하단에 오려면 footer를 감싸는 div는 relative로 설정하고 min-height:100% footer는 absolute로 설정,html과 body의 height도 100% 설정하니 되더라```
* config/settings.py (base.html경로 추가)
```
TEMPLATES=[{
    'DIRS':[os.path.join(BASE_DIR,"templates")],
}]
```
* photo/list.html
```
{% extends 'base.html' %}
{% block title %}-List{% endblock %}

{% block content %}
    {% for post in photos %}
    <div class="row">
        <div class="col-md-2">
        </div>
        <div class="col-md-8 panel panel-default">
            <p><img src="{{post.photo.url}}" style="width:100%;"></p>
            <button type="button" class="btn btn-xs btn-info">
                {{post.author.username}}
            </button>
            <p>{{post.text|linebreaksbr}}</p>
            <p class="text-right">
                <a href="{% url 'photo:photo_detail' pk=post.id %}" class="btn btn-xs btn-success">댓글달기</a>
            </p>
        </div>
        <div class="col-md-2"></div>
    </div>
    {% endfor %}
{% endblock %}
```
1. 템플릿으로 전달되는 사진 오브젝트 목록을 photos라는 변수명으로 설정했었음
2. div class=row,col... :부트스트랩에서 테이블 형식으로 row,col 클래스들을 자유롭게 구사하고 알아서 크기에 따라 반응형으로 동작   
* 규칙```row클래스는 container안에 있어야 정상적인 배열이나 패팅을 지원 / row 클래스는 가로로 그룹 지울 칼럼들의 집합 / 내용은 col-*클래스 안에 있어야 하며 row 직속 자녀 요소로 배치 / 칼럼은 총12칼럼이 있는것으로 정의하여 각 배치할 %에 따라서 클래스를 결정 /12칼럼이 넘어가면 새로운 줄로 칼럼 배치```   
* 칼럼 클래스 종류```col-xs :항상 가로로 배치(모바일 폰) / col-sm :768px이하에서 세로로 표시 시작(태블릿) / col-md : 992px이하에서 세로로 표시 시작 / col-lg : 1200px이하에서 세로로 표시 시작(데스크탑)```
3. panel panel-default :패널은 콘텐츠가 있는 박스 형태의 구성요소를 만들때 사용 /panel-default는 기본 색상이다 / primary,successs,warning,info등등이 있다..
4. 이미지 주소 출력할때는 photo.url을 사용한다
5. 사용자명은 author.username을 사용한다
6. linebreaksbr 속성은 줄바꿀때 사용
7. 댓글달기 누르면 상세페이지로 이동 
8. class="text-right" 부트스트랩에서 오른쪽으로 정렬해줌..
* photo/upload.html
```
{% extends 'base.html' %}
{% block title %}-Upload{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-2"></div>
    <div class="col-md-8 panel panel-default">
        <form action="" method="POST" enctype="multipart/form-data">
            {{form.as_p}}
            {% csrf_token %}
            <input type="submit" class="btn btn-primary" value="Upload">
        </form>
    </div>
    <div class="col-md-2"></div>
</div>
{% endblock %}
```
enctype :form태그로 작성한 정보르 어떤 형태로 인코딩해서 서버로 전달할 것인지 결정하는 옵션 /method가 POST일때만 사용가능   
옵션```applicationx-www-form-urlencoded: 기본옵션,모든 문자열을 인코딩해 전달 multipart/form-data:파일 업로드때 사용하는 옵션 , 데이터를 문자열로 인코딩 하지 않고 전달등..```
* photo/detail.html
```
{% extends 'base.html' %}
{% block title %}{{object.text|truncatechars:10}}{% endblock %}

{% block content %}
    <div class="row">
        <div class="col-md-2"></div>
        <div class="col-md-8 panel panel-default">
            <p><img src="{{object.photo.url}}" style="width:100%;"></p>
            <button type="button" class="btn btn-outline-primary btn-sm">{{object.author.username}}</button>
            <p>{{object.text|linebreaksbr}}</p>
            <a href="{% url 'photo:photo_delete' pk=object.id %}" class="btn btn-outline-danger btn-sm float-right">Delete</a>
            <a href="{% url 'photo:photo_update' pk=object.id %}" class="btn btn-outline-success btn-sm float-right">Update</a>
            {% load disqus_tags %}
            {% disqus_show_comments %}
        </div>
        <div class="col-md-2"></div>
    </div>
{% endblock %}
```
1. truncatechars: 글자수 자르기
2. class="float-right" : 오른쪽에 배치
* photo/delete.html
```
{% extends 'base.html' %}
{% block title %}-Delete{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-2"></div>
    <div class="col-md-8 panel panel-default">
        <div class="alert alert-info">
            Do you want to delete {{object}}?
        </div>
        <form action="" method="POST">
            {{form.as_p}}
            {% csrf_token %}
            <input type="submit" class="btn btn-danger" value="Confirm">
        </form>
    </div>
    <div class="col-md-2"></div>
</div>
{% endblock %}
```
## 9.사진 표시하기 (지금까지는 사진이 출력 안됐음)
* config/urls.py
```
from django.conf.urls.static import static
from django.conf imort settings

urlpatterns+=static(settings.MEDIA_URL,document_root=settings.MEDIA_ROOT)
```
DEBUG=='True'일때만 동작 

# Account 앱 만들기
## 1.accounts앱 만들기
1. python manage.py startapp accounts
* config/settings.py
```
INSTALLED_APPS=[
    'accounts' //추가
]
```

## 2.로그인,로그아웃 기능 추가
로그인,로그아웃은 장고에 이미 만들어져있는 기능이다 이 기능을 불러서 사용하면 된다
* accounts/urls.py (기존에 있는 뷰를 불러서 사용)
```
from django.urls import path
from django.contrib.auth import views as auth_view

urlpatterns=[
    path('login/',auth_view.LoginView.as_view(),name='login'),
    path('logout/',auth_view.LogoutView.as_view(template_name='registration/logout.html'),name='logout')
]
```
기본 템플릿 이름이 logged_out.html이라 이름 바꿔줌
* config/urls.py (루트 urls.py에 연결)
```
path('accounts/',include('accounts.urls'))
```
* registration/login.html
```
{% extends 'base.html' %}
{% block title %}-Login{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-2"></div>
    <div class="col-md-8 panel panel-default">
        <div class="alert alert-info">Please enter your informations.</div>
        <form action="" method="POST">
            {{form.as_p}}
            {% csrf_token %}
            <input class="btn btn-primary" type="submit" value="Login">
        </form>
    </div>
    <div class="col-md-2"></div>
</div>
{% endblock %}a
```
* registration/logout.html
```
{% extends 'base.html' %}
{% block title %}-Logout{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-2"></div>
    <div class="col-md-8 panel panel-default">
        <div class="alert alert-info">You have been succesfully logged out.</div>
        <a class="btn btn-primary" href="{% url 'login' %}">Click to Login</a>
    </div>
    <div class="col-md-2"></div>
</div>
{% endblock %}
```
* base.html에서 로그인,로그아웃 연결
```
<li class="nav-item"><a href="{% url 'logout' %}" class="nav-link">Logout</a></li>
<li class="nav-item"><a href="{% url 'login' %}" class="nav-link">Login</a></li>
```
로그인 후 이동할 페이지 설정 기본값이 profile임 -> profile을 만들지 않았기때문에 오류 발생
* config/settings.py (로그인 후 메인페이지로 이동하도록 변경)
```
LOGIN_REDIRECT_URL='/'
```
## 3.회원가입 기능 만들기
회원 가입 기능을 만들기 위해서는 뷰와 폼이 필요하다
* accounts/forms.py
```
from django.contrib.auth.models import User
from django import forms

class RegisterForm(forms.ModelForm):
    password=forms.CharField(label="Password",widget=forms.PasswordInput)
    password2=forms.CharField(label="repeat Password",widget=forms.PasswordInput)
    class Meta:
        model=User
        fields=['username','first_name','last_name','email']
    def clean_password2(self):
        cd=self.cleaned_data
        if cd['password']!=cd['password2']:
            raise forms.ValidationError('Passwords not matched')
        return cd['password2']
```
1. 회원 가입 양식을 출력하기 위해 Registerform이라는 클래스 생성
2. forms.ModelForm 상속 -> 모델이 있고 그에 대한 자료를 입력받고 싶을 때 사용/ 모델과 필드를 지정하면 모델폼이 자동으로 폼 필드를 생성 함
3. Meta 클래스 : 입력 받을 필드 설정 /password는 widget 옵션을 사용해 input type을 password로 바꾸려고 클래스 변수로 따로 지정
4. clean_password2 메소드 :   
* clean_필드명->clean 메소드가 호출된 후에 호출되는 메소드 (유효성 검사나 조작을 하고 싶을때 만들어 사용)
* cleaned_data에서 필드값을 찾아서 사용해야 한다 -> 이전 단계까지 기본 유효성 검사같은 처리를 마친 값이기 때문

* accounts/views.py
```
from django.shortcuts import render
from .forms import RegisterForm

def register(request):
    if request.method=='POST':
        user_form=RegisterForm(request.POST)
        if user_form.is_valid():
            new_user=user_form.save(commit=False)
            new_user.set_password(user_form.cleaned_data['password'])
            new_user.save()
            return render(request,'registration/register_done.html',{'new_user':new_user})
    else:
        user_form=RegisterForm()
    return render(request,'registration/register.html',{'form':user_form})
```
1. request.method=='POST' :post방식으로 뷰를 호출했다 -> 서버로 자료를 전달 함
2. 유저 정보가 타당하면->   
user_form.save메소드 : 폼 객체에 지정된 모델을 확인하고 이 모델의 객체를 만듦(commit=False->데이터 베이스에 저장하는 것이 아니라 메모리 상에 객체만 만들어짐)   
set_password메소드 :비밀번호 암호화된 상태로 저장   
save 메소드 호출해 데이터베이스에 저장하고 regitser_done템플릿 랜더링
3. http가 POST가 아니면 아직 자료를 전달받은 상태가 아니기 때문에 비어있는 RegisterForm 객체를 만들고 register템플릿 랜더링   
* accounts/urls.py (register 뷰를 연결)
```
from .views import register
urlpatterns=[
    path('register/',register,name='register'),
]
```
* registration/register.html (회원가입 페이지)
```
{% extends 'base.html' %}
{% block title %}-Registration{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-2"></div>
    <div class="col-md-8 panel panel-default">
        <div class="alert alert-info">Please enter your account informations.</div>
        <form action="" method="POST">
            {{form.as_p}}
            {% csrf_token %}
            <input class="btn btn-primary" type="submit" value="Register">
        </form>
    </div>
    <div class="col-md-2"></div>
</div>
{% endblock %}
```
* registration/register_done.html (회원가입 완료 페이지)
```
{% extends 'base.html' %}
{% block title %}-Registration Done{% endblock %}

{% block content %}
<div class="row">
    <div class="col-md-2"></div>
    <div class="col-md-8 panel panel-default">
        <div class="alert alert-info">Registration Success. Welcome,{{new_user.username}}</div>
        <a class="btn btn-info" href="/">Move to main</a>
    </div>
    <div class="col-md-2"></div>
</div>
{% endblock %}
```
* base.html에 회원가입 링크 연결
```
<a href="{% url 'register' %}" class="nav-link">Signup</a>
```

# 댓글 기능 구현하기
DISQUS라는 온라인 소셜 댓글 시스템을 빌려서 사용
1. disqus 회원가입->무료 요금제 선택
2. Disqus 앱 설치   
pip install django-disqus
3. settings.py에 등록   
```
INSTALLED_APPS=[
    'disqus',
    'django.contrib.sites', //장고에서 사용하는 사이트 관리 프레임워크
]
```
4. sites앱을 위한 데이터베이스 설정    
python manage.py migrate
5. disqus사용을 위한 설정값 추가   
* config/settings.py
```
DISQUS_WEBSITE_SHORTNAME='disqus에 설정해둔 이름'
SITE_ID=1 //sites앱에 등록된 현재 사이트 번호 (기본적으로 1이다)
```
6. 상세 페이지에서 댓글이 보이도록 detail템플릿 수정   
* photo/detail.html
```
{% load disqus_tags %}
{% disqus_show_comments %} //추가
```
7. 권한 제한하기 (로그인 한 사용자만 Dstagram 서비스 이용 가능하게끔)   
->데코레이터와 믹스인 사용
* photo/views.py
```
from django.contrib.auth.decorators import login_required //함수형 뷰에서 사용
from django.contrib.auth.mixins import LoginRequiredMixin //클래스형 뷰에서 사용

@login_required //뷰의 바로 윗줄에 써줌
def photo_list(request) 

class PhotoUploadView(LoginRequiredMixin,Createview): //믹스인 첫번째로 상속
class PhotoDeleteView(LoginRequiredMixin,DeleteView):
class PhotoUpdateView(LoginRequiredMixin,UpdateView):
```

