> <br>
>
> ## **Overview**
>
> <br>
>
> - [**영속성 컨텍스트란 ?**](#영속성-컨텍스트란)
>
> - [**엔티티의 생명주기**](#엔티티의-생명주기)
>
>   - [비영속(new / tansient)](#비영속)
>   - [영속(managed)](#영속)
>   - [준영속(detached)](#준영속)
>   - [삭제(removed)](#삭제)
>
> - [**영속성 컨텍스트 장점**](#영속성-컨텍스트-장점)
>
> <br>

<br />
<br />
<br />
<br />
<br />

# **영속성 컨텍스트란 ?**

- 엔티티를 관리하고 저장하는 환경
- Jpa 어플리케이션과 Database 사이에서 객체를 관리하는 논리적 개념

# **엔티티의 생명주기**

#### **비영속 (new / transient)**

- 영속화 되기 전의 상태
- 순수한 Entity 객체
- EntityManager 로부터 관리받지 않는 상태

```java

```

<hr>

#### **영속 (managed)**

- EntityManager로 부터 관리받는 상태
- 영속성 컨텍스트의 장점을 사용할 수 있음 ex) 1차 캐시 , Eager or Lazy Loading ...
- asd

<hr>

#### **준영속 (detached)**

<hr>

#### **삭제 (removed)**

<hr>

<br>
<br>

# **영속성 컨텍스트 장점**
