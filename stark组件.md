### stark组件

**1.注册**

在settings配置文件中INSTALLED_APPS[]添加app（stark）
**2.启动**

2.1在stark.apps文件下增加ready()方法，并导入django.utils.module_loading调用autodiscover_modules()方法

```python
from django.apps import AppConfig
from django.utils.module_loading import autodiscover_modules


class StarkConfig(AppConfig):
    name = 'stark'

    def ready(self):
        # django一启动会扫描每一个叫stark的文件
        autodiscover_modules('stark')
# 注：autodiscover_modules()方法的作用是django一启动会扫描每一个叫stark的文件
```

2.2 在stark下建立service 包，并在其下建立stark.py文件

```python
# 配置类
class ModelStark(object):
    def __init__(self, model, site):
        self.model = model
        self.site = site

# 单例模式
class StarkSite(object):
    def __init__(self):
        self._registry = {}
	
    # 注册函数
    def register(self, model, stark_class=None):
        if not stark_class:
            stark_class = ModelStark
        self._registry[model] = stark_class(model, self)


site = StarkSite()
```

**3.设计url**

3.1在项目urls.py中导入stark的url

```python
from stark.service import stark
from django.urls import path


urlpatterns = [
    path('stark/', stark.site.urls),
]
```

3.2在stark的service 包下的stark.py文件增加url一级分发

```python
from django.urls import path
from django.shortcuts import HttpResponse, render


class ModelStark(object):
    def __init__(self, model, site):
        self.model = model
        self.site = site

        
class StarkSite(object):
    def __init__(self):
        self._registry = {}
        
	# 注册函数
    def register(self, model, stark_class=None):
        if not stark_class:
            stark_class = ModelStark
        self._registry[model] = stark_class(model, self)
        
    # 一级分发视图    
    def view(self, request):
        return HttpResponse('view')
	
    def get_urls(self):
        temp = []
        for model, stark_class_obj in self._registry.items():
            mode_name = model._meta.model_name
            app_label = model._meta.app_label
            temp.append(path('{0}/{1}/'.format(app_label, mode_name,), self.view))
        return temp

    @property
    def urls(self):
        return self.get_urls(), None, None


site = StarkSite()
```

3.2在stark的service 包下的stark.py文件增加url二级分发

```python
from django.urls import path
from django.shortcuts import HttpResponse, render


class ModelStark(object):
    def __init__(self, model, site):
        self.model = model
        self.site = site

    def list_view(self, request):
        model_list = self.model.objects.all()
        return render(request, 'list_view.html', locals())

    def add_view(self, request):
        return HttpResponse('add_view')

    def change_view(self, request, id):
        return HttpResponse('change_view')

    def delete_view(self, request, id):
        return HttpResponse('delete_view')

    def get_urls2(self):
        temp = []
        temp.append(path('', self.list_view))
        temp.append(path('add/', self.add_view))
        temp.append(path('<int:id>/change/', self.change_view))
        temp.append(path('<int:id>/delete/', self.delete_view))
        return temp

    @property
    def urls2(self):
        return self.get_urls2(), None, None


class StarkSite(object):
    def __init__(self):
        self._registry = {}
        
	# 注册函数
    def register(self, model, stark_class=None):
        if not stark_class:
            stark_class = ModelStark
        self._registry[model] = stark_class(model, self)
	
    def get_urls(self):
        temp = []
        for model, stark_class_obj in self._registry.items():
            mode_name = model._meta.model_name
            app_label = model._meta.app_label
            # 为便于二级分发每个视图可定制，需通过配置类实现二级分发功能
            temp.append(path('{0}/{1}/'.format(app_label, mode_name,), stark_class_obj.urls2))
        return temp

    @property
    def urls(self):
        return self.get_urls(), None, None


site = StarkSite()
```

