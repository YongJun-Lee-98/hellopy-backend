# HelloPY - Backend

- 개발기간 : 3개월  
- 참여인원 : \[Backend : 3] \[Frontend : 3] \[ProductManager : 2] \[Designer : 2]  
- 사용기술 : <img src="https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=Python&logoColor=yellow"/>, <img src="https://img.shields.io/badge/Django-092E20?style=flat-square&logo=Django&logoColor=yellow"/>  
Django REST framework, Django Ckeditor 5, Django Jazzmin, Django Filter, BeautifulSoup4  
- ERD : [링크](https://dbdiagram.io/d/hellopy-backend-67b2c379263d6cf9a05f4c3b)



> 회고
### 코드 방식
Model class를 abstract(추상)클래스로 만들어 동일한 기능을 여러번 정의하지 않고 객체지향의 상속 방식을 활용해보는 기회가 되었습니다.
기초적인 Django 입문서를 보면 "class modelname(models.Model):" model 의 클래스를 정의하게되는데 공통된 값들 혹은 설정들을 묶어서 상속하도록 표현하고
정의한 class는 abstract 로 정의하여 Django가 makemigrations 명령어를 입력하였을 때  
해당 모델은 abstract 이므로 DB에 TABLE을 생성하지 않는다는 점을 활용할 수 있었습니다.  

### 트러블 슈팅
[Github issue](https://github.com/HelloPy-Korea/hellopy-backend/issues) 를 활용한 트러블 슈팅
#### 문제상황
Django의 admin 페이지에서 데이터를 삭제 시,  
ImageField와 CKEditor에서 업로드된 이미지 파일들이 그대로 남아  
삭제되지 않는 문제의 발생을 팀원이 발견하여 issue에 등록하였고  

#### 원인분석
1. ImageField 의 원인
 ImageField의 data가 삭제되지 않는 이유는 django에서 delete에 기본적으로 정의되지 않아서였습니다.
django > db > models > base.py > class Model에서 다음을 확인할 수 있었습니다.
   ```python
       def delete(self, using=None, keep_parents=False):
        if self.pk is None:
            raise ValueError(
                "%s object can't be deleted because its %s attribute is set "
                "to None." % (self._meta.object_name, self._meta.pk.attname)
            )
        using = using or router.db_for_write(self.__class__, instance=self)
        collector = Collector(using=using, origin=self)
        collector.collect([self], keep_parents=keep_parents)
        return collector.delete()

    delete.alters_data = True
   ```
2. Django-Ckeditor-5의 원인
원인 자체는 ImageField와 비슷하게 정의에 있었습니다.  
다만, Django-Ckeditor-5는 storage(저장소)에 대한  
정의가 잘못되었다는 것을 발견할 수 있었습니다.  
```python
@receiver(pre_delete)
def cleanup_ckeditor_images_on_delete(sender, instance, **kwargs):
    """
    Removes images from disk when an object is deleted.
    If an error occurs, it is logged, but the deletion process continues.
    """
    try:
        storage = get_storage_class()
        if not storage:
            return

        # Find CKEditor5Field dynamically
        images_to_delete = []
        for field in instance._meta.fields:
            if isinstance(field, CKEditor5Field):
                images_to_delete.extend(
                    extract_image_paths(getattr(instance, field.name, ""))
                )
        try:
            delete_images(storage, images_to_delete)
```
정확히 잘못된 부분은 해결과정에서 설명드리겠습니다.  

#### 해결과정
1. ImageField의 해결과정
ImageField의 경우 여러 apps에서 동일한 정의를 사용하여 이미지를 받아오고 삭제할 수 있어야 했습니다.
따라서, public/mixin 내부에 img_models.py를 두어 abstract 모델을 정의하였습니다.
```python
def delete(self, *args, **kwargs):
    image_field = getattr(self, self.image_field_name, None)
    if image_field and hasattr(image_field, "path"):
        image_path = image_field.path
        if os.path.exists(image_path):
            os.remove(image_path)
    super().delete(*args, **kwargs)
```
주요 내용은 delete가 호출되면 DB에 저장된 필드의 path를 가져오고  
해당 path를 os 모듈을 통해 존재하는지 확인한 뒤  
존재한다면 삭제한 뒤 기존 django가 기본적으로 실행하는 delete()함수를 실행하도록 수정하였습니다.  
  
2. Django-Ckeditor-5의 해결과정
Ckeditor-5의 경우 라이브러리상의 문제여서 라이브러리 자체를 수정하는 것은 근본적인 해결방안이 될 수 없었습니다.
로컬에서 작업되는 것이 아닌 서버에 배포를 해야함으로 두 가지의 방향으로 문제를 해결하고자 했습니다.
- models.py에서 delete()를 직접 정의
- Django-Ckeditor-5에 delete()함수를 수정한 code contribute

delete() 정의의 경우  
1번과 결이 비슷한 해결방법입니다.  
BeautifulSoup4를 활용하여 이미지 주소를 파싱한 뒤  
파싱한 주소를 os 라이브러리로 삭제한다는 생각입니다.  
```python
def delete(self, *args, **kwargs):
        self._delete_ckeditor_images()
        super().delete(*args, **kwargs)

    def _delete_ckeditor_images(self):
        soup = BeautifulSoup(self.content, "html.parser")
        for img_tag in soup.find_all("img"):
            src = img_tag.get("src")
            if src and src.startswith(settings.MEDIA_URL + "notice/ckeditor"):
                relative_path = src.replace(settings.MEDIA_URL, "")
                file_path = os.path.join(settings.MEDIA_ROOT, relative_path)
                if os.path.isfile(file_path):
                    os.remove(file_path)
```

다음은 code contribute의 경우입니다.  
앞서 설명한 부분은 현재 프로젝트에 적용하기 위한 코드였습니다.  
제가 코드를 수정해서 PR을 올린다고 해도 바로 받아들여지기는 어려울거라 판단했습니다.  
해당 코드들이 직접 정의된 곳과 적용된 곳을 확인해 본 결과  

storage = get_storage_class() 부분이 인스턴스를 참조하고 있는 것이 아니라  
class를 직접 참조하고 있었습니다.  

storage는 결과적으로 os 모듈로 join하여 이미지의 path를 구한다는 생각이였지만  
os의 join이 폭 넓게 주소를 합쳐준다 하더라도 값이 str이 아닌  
인스턴스화 하지 않은 class를 join 해주진 못하였습니다.  

따라서 해결방법 자체는 간단하였습니다. 
```python
storage_class = get_storage_class()
storage = storage_class()
```
로 수정하여 다른 코드의 변화 없이 storage라는 변수가  
class의 인스턴스를 참조하도록 해주니 정상적으로 delete가 작동하였습니다.  
이를 바탕으로 해당 라이브러리에 PR을 올려두었습니다.  
[Django-CKEditor-5 PR](https://github.com/hvlads/django-ckeditor-5/pull/293)  

#### 회고
models를 정의하는데 있어 CRUD가 일어날 때  
Django의 정의를 좀 더 깊이 이해할 수 있었으며  
abstract를 활용하는 부분과 Django의 models의 정의를 결합하여   
기본 Model.models의 save, delete, clean을 더 잘 활용할 수 있게되었습니다.
