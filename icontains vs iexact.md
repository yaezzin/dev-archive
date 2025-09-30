# icontains vs iexact

## 1. iexact

```python
Model.objects.get(name__iexact='example')
Model.objects.get(name__iexact=None)
```

```sql
SELECT ...
FROM ...
WHERE UPPER(name) = UPPER("example")
```

* `iexact` : 특정 필드 값이 대소문자를 구분하지 않고 **정확히** 일치하는 레코드를 찾을 때 사용

## 2. icontains

```python
Model.objects.get(name__icontains='example')
```

```sql
SELECT ...
FROM ...
WHERE name ILIKE '%example%'
```
* `icontains` : 특정 필드 값이 대소문자를 구분하지 않고 특정 문자열을 포함하는 레코드를 찾을 때 사용

## 3. 인덱싱

📌 `db_index=True`

```python
# example
class Person(models.Model):
    name = models.CharField(max_length=100, db_index=True)
```
* Django 모델 필드에 db_index=True 설정 시 기본적으로 PostgreSQL에서는 B-Tree 인덱스 생성됨
* B-Tree의 경우 비교 연산자에는 효율적이나 대소문자를 구분하지 않는 패턴 매칭(LIKE, ILIKE)에는 기본적으로 활용되지 않음 ➡️ 따라서 `field__iexact='value'`와 같은 쿼리는 B-Tree 인덱스를 타지 못함
* Django는 Like 쿼리 최적화를 위해 varchar_pattern_ops를 사용하는 두 번째 인덱스를 생성하나, 이 인덱스도 문자열의 시작 'like%'에만 작동함 (%like% 연산 같이 데이터 중간에 속한 값에 대한 인덱스 적용 불가)
  

📌 `Gin Index`

* Gin Index는 like 검색이 어려운 B-Tree 문제점을 해결해줄 수 있는 full text search를 위해 만들어진 인덱스
* 인덱스를 적용하는 컬럼의 값을 일정 규칙에 따라 쪼개고, 쪼갠 요소들을 활용하는 방식 (단어를 해당 테이블 위체에 매핑시켜 역순으로 데이터를 찾아감)

```python
class Migration(migrations.Migration):
    dependencies = [
        # ... 이전 마이그레이션 ...
    ]
    operations = [
        # Trigram 확장 설치
        TrigramExtension(),
        # ... 여기에 GinIndex 추가 마이그레이션 ...
    ]
```
* django.contrib.postgres.indexes.GinIndex
* GinIndex를 텍스트 검색 성능 향상에 사용하려면 PostgreSQL의 Trigram 확장과 함께 사용하는 것이 일반적임

```python
class Meta:
    indexes = [
        GinIndex(fields=['name'], name='name_gin_index', opclasses=['gin_trgm_ops'])
    ]
```
* 모델의 Meta 클래스 내 indexes 리스트에 GinIndex를 추가하고, 텍스트 검색을 위해 opclasses=['gin_trgm_ops']를 반드시 지정해야함


## 다시 읽어보면 좋을 글들
* [Gin Index를 이용한 Like 검색 성능 개선(with PostgreSQL Index)](https://medium.com/@heeee/django-gin-index%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-like-%EA%B2%80%EC%83%89-%EC%84%B1%EB%8A%A5-%EA%B0%9C%EC%84%A0-with-postgresql-index-9c9eae7f67b7)



