---
title: Spring Security 6에서 CustomUserDetailsService 이용하기
lang: ko
layout: post
---

## 들어가며

### Disclaimer

- Last Update: 2024/08/11

- SSAFY 내 이전 프로젝트에서 WebSocket을 맡았으나, 인원 부족으로 인한 팀 해체로 새로운 팀에서 Spring Security를 이용한 기존 인증과 인가 코드를 수정하고 개선하는 일에 우선적으로 돌입했습니다.
- 이전 코드는 컨트롤러 기반의 로그인이 아니라, Fitler 위주로 JWT Filter 및 Login Filter를 이용하고 있었습니다.
- Spring Security를 처음 이용하며, 마주하는 전체적인 그림이나 트러블슈팅에 대해 자주 글을 쓸 예정입니다.

<br/>

## 문제 상황

### 테스트 코드가 왜 UserDetailsService를 이용할까?

이번 SSAFY 프로젝트에서는 TDD(Test Driven Development)까지는 아니더라도, JUnit과 Mockito를 이용해 테스트 코드를 주의를 기울여 작성해보고 싶었습니다. 특히, 단위 테스트를 이용해 개별 기능으로부터 얼마나 책임이 잘 분리되어 있는지도 고민해보고 싶었는데요. Spring Security의 넓은 의존성을 관리하거나 관련된 설정을 적용하는 것은 다른 글로 쓰고, 이번 글에서는 다음 예외에서 출발합니다.

> org.springframework.security.authentication.InternalAuthenticationServiceException: UserDetailsService returned null, which is an interface contract violation

이 예외는 Spring Security 내에서 제공되는 `UserDetailsService` 인터페이스 구현체에서 `loadUserByUserName()`을 통해 유저 정보를 찾지 못했을 때 던져집니다. 여기에서 `UserDetailService` 궇녀체라고 함은 `AuthenticationProvider` 구현체에서 인증을 위해 이용하는 서비스 클래스라고 볼 수 있습니다. 하지만 테스트 코드를 통한 디버깅 결과, 따로 만든 `CustomUserDetailsService`를 이용하지 않고 있다는 점이 더욱 문제적이었습니다.

깊게 들어가기 전에! Spring Security를 이제 막 시작하시는 분이라면, 아래 살펴보기를 읽고 가셔도 좋겠습니다.

### 살펴보기: CustomUserService에 대해서

사실 직접 명명한 이름인 CustomUserDetailsService라고 부르기보다는, Spring Security에서 제공하는 `UserDetailsService` 인터페이스의 구현체 클래스라고 말하는 것이 옳을지도 모르겠습니다. 이름을 하나씩 뜯어보자면, Custom + UserDetails + Service라고 볼 수 있는데요. Service가 서비스 레이어를 의미한다면, Custom은 요구에 맞게 변경했다는 뜻이겠습니다. 여기에서 중요한 것은 `UserDetails`입니다.

Spring Security에서 `UserDetails` 인터페이스는 권한과 관련된 `getAuthorities()`, 주로 비밀번호로 대표되는 credential과 관련된 `getPassword()`, 그리고 유저의 구분자가 되는 유저네임과 관련된 `getUsername()` 메서드가 있습니다. 물론, 해당 유저의 상태를 나타내는 디폴트 메서드들 역시 오버라이드 해야합니다. 그런데 제가 앞서 "유저의 구분자"라는 표현을 썼었죠? 실제로 `UserDetails` 인터페이스를 구현한 Spring Security 내 `User` 클래스는 다음과 같이 username과 관련된 코드를 구성하고 있습니다.

```java
private final String username;
...
public User(String username, String password, Collection<? extends GrantedAuthority> authorities) {
    this(username, password, true, true, true, true, authorities);
}

public User(String username, String password, boolean enabled, boolean accountNonExpired, boolean credentialsNonExpired, boolean accountNonLocked, Collection<? extends GrantedAuthority> authorities) {
  Assert.isTrue(username != null && !"".equals(username) && password != null, "Cannot pass null or empty values to constructor");
    this.username = username;
    this.password = password;
    this.enabled = enabled;
    this.accountNonExpired = accountNonExpired;
    this.credentialsNonExpired = credentialsNonExpired;
    this.accountNonLocked = accountNonLocked;
    this.authorities = Collections.unmodifiableSet(sortAuthorities(authorities));
}

...
public String getUsername() {
    return this.username;
}

```

제가 앞서 '유저명'이 아니라 구분자라는 표현을 쓴 이유는, Spring Securtiy에서 인증이 이루어지는 과정과 관련이 있습니다. (추후에 Spring Security의 전체적인 인증 및 인가 과정을 다뤄보겠습니다.) `AuthenticationManager` 인터페이스 구현체는 인증을 위해서 `Authentication`을 받기도 하고, 인증 이후에 `SecurityContextHolder`에 `Principal`을 등록할 대에도 `Authentication` 구현체를 보냅니다. 전자의 `Authentication`은 유저 구분자와 비밀번호를 이용하고, 후자의 `Authentication`은 `Principal`과 `GrantedAuthority`를 이용합니다.

그리고 CustomUserDetailsService에서 오버라이드된 `loadUserByUsername()` 메서드는 `MemberRepository`에서 email로 Member 엔티티를 조회합니다.

## 문제 해결

### Spring Security 6에서의 설정 (1): 코드 뜯어보기

당연하게도, 가장 먼저 의심이 되는 쪽은 Spring Security 설정 관련 코드가 있는 `SecurityConfig.java` 파일이었습니다. 팀에 합류하기 전 기존에 작성되어 있던 코에서는 당연히 설정이 완벽하게 되어 있겠지, 하고 주의를 덜 기울였던 저를 반성하게 되었습니다. 실제로 `CustomUserDetailsService`는 전혀 이용되고 있지 않았습니다.

주목해야 하는 컴포넌트는 `AuthenticationManager` 인터페이스 구현체입니다. `AuthenticationManager`는 `UserDetailsService` 인터페이스 구현체에 의존성이 있습니다. 인증 과정에서는 요청에서 전달받은 username을 가진 유저의 여러 속성에 기반해 검증하기 때문인데요. 여기에서 따로 `AuthenticationManager`나 `AuthenticationProvider`를 처리해 생성하지 않는다면 기존에 Spring Security에서 제공되는 `UserDetailsService`를 이용하게 됩니다. 그렇다면 어떻게 `AuthenticationManager`가 `CustomUserDetails`를 이용할 수 있을까요?

저는 검색보다도 직접 코드를 뜯어보면서 문제 해결의 실마리를 발견할 수 있었습니다. 아래는 일차적으로 해결된 코드입니다.

```java
// SecurityConfig.java
// 코드 (1)

@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration configuration) throws Exception {
    List<GlobalAuthenticationConfigurerAdapter> configurerAdapterList = new ArrayList<>();
    GlobalAuthenticationConfigurerAdapter globalAuthenticationConfigurerAdapter = new GlobalAuthenticationConfigurerAdapter() {
        @Override
        public void configure(AuthenticationManagerBuilder auth) throws Exception {
            auth.userDetailsService(customUserDetailsService);
            super.configure(auth);
        }
    };
    return configuration.getAuthenticationManager();
}

```

Spring Security 5.4 버전 이후로는 더 이상 Config 파일에서 `WebSecurityConfigurerAdapter`를 상속하지 않습니다. 가장 큰 차이를 꼽자면, SecurityFilterChain을 Bean으로 올린다는 점이 되겠는데요. 그에 따라 `AuthenticationManager`도 `configure(AuthenticationManagerBuilder auth)` 메서드를 오버라이드해 이용하지 않게 되었습니다.

`authenticationManager(AuthenticationConfiguration)` 메서드는 팀에 합류할 때부터 존재했습니다. 저는 이 코드를 최대한 변경하지 않고 작성하고 싶었습니다. 그래서 `AuthenticationConfiguration`의 코드를 분석하며 내부에서 주입된 의존성 객체들이 어떻게 `AuthenticationManager`를 구성하게 되었는지 살폈습니다.

결론적으로는 `AuthenticationManagerBuilder`를 핵심으로 삼고, 이를 이용해 `userDetailService`를 설정하게 두었는데요. 하지만 이 코드는 `AuthenticationConfiguration`을 이용하나,`GlobalAuthenticationConfigrurerAdapter` 타입에서도 보이듯 Spring Security의 레거시 코드 냄새가 납니다. 과연 이 코드가 Spring Security의 걸맞는 좋은 코드라고 할 수 있을까요?

### Spring Security 6에서의 설정 (2): AuthenticationProvider 이용하기

저는 Spring Security에서의 인증 흐름에 따라, `AuthenticationManager` 구현체가 `AuthenticationProvider` 구현체를 이용한다는 점을 코드에 명시하고 싶었습니다. (1) 코드 뜯어보기에서 쓰인 코드는 `AuthenticationConfiguration`을 이용하긴 하지만, 실제로는 `configure()` 메서드를 오버라이드해 진행한다는 점에서 `WebSecurityConfigurerAdapter`를 쓰는 것과 큰 차이가 없는 까닭입니다. 게다가 저처럼 Spring Security를 처음 다루면서도, 기존 회원 관리 코드를 수정하고 새로 다루는 인원이 코드를 본다면 어떻게 될까요? 따라서 코드를 아래와 같이 수정했습니다.

```java
// 코드 (2)

public DaoAuthenticationProvider authenticationProvider() {
		DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
		authProvider.setUserDetailsService(customUserDetailsService);
		authProvider.setPasswordEncoder(bCryptPasswordEncoder);
		return authProvider;
	}

@Bean
public AuthenticationManager authenticationManager(AuthenticationConfiguration authenticationConfiguration) throws Exception {
    // authenticationManagerBuilder는 생성자 주입
    authenticationManagerBuilder.authenticationProvider(authenticationProvider());
    return authenticationConfiguration.getAuthenticationManager();
}
```

`AuthenticationManagerBuilder`를 통해서 `authenticationManager`가 어떤 `authenticationProvider`를 이용할지 정해둘 수 있는데요. `authenticationProvider()` 메소드를 통해 `DaoAuthenticationProvider` 구현체를 생성하고 해당 구현체에서 `customUserDetails`와 `BCrpytPasswordEncoder`를 이용하도록 설정한 뒤 해당 구현체를 반환하도록 합니다. `DaoAuthenticationProvider`는 `Authentication` 구현체 중 하나인 `UsernamePasswordAuthenticationToken`을 이용할 때 인증 작업을 진행하는 대표적인 `AuthenticationProvider` 구현체라는 점에서 알맞다고 판단했습니다.

### 어떤 게 더 좋은 코드일까?

이전에 `customUserDetailsService`를 이용하지 않았던 문제는 차치하고, (1)과 (2)만 비교해보겠습니다. 이해도와 책임이라는 두 기준 아래에서, 어떤 코드가 더 나을까요?

- 이해도

코드 (1)에서는 `GlobalAuthenticationConfugrerAdapter`를 이용해 추가적인 설정을 진행할 수 있는 확장성이 있지만, 현재 프로젝트 레벨에서는 불필요합니다. 게다가 다른 `UserDetailsService` 구현체를 이용하고 싶을 때는 다른 `AuthenticationProvider` 구현체를 이용해야하지만, 이 코드에서는 이러한 흐름이 드러나지 않습니다. 즉, 단순히 `customUserDetailsService`를 이용하기 위해 작성한 코드이지만 새롭게 코드를 이해해야하는 입장에서는 `GlobalAuthenticationConfigurerAdapter`가 무엇인지부터 살펴보기 쉽지 않습니다. 게다가 익명 클래스를 이용해 메서드를 오버라이드하면서, 원래 `configure()`이 어떠한 위치를 차지하는 메서드인지도 파악해야 합니다.

반면, 코드 (2)에서는 `AuthenticationManager`가 `AuthenticationProvider`를 구현하고 있음이 명확히 드러납니다. `AuthenticationProvider`에서는 특정 `UserDetailsService` 구현체와 `PasswordEncoder` 구현체를 이용한다는 점을 알 수 있고요. 이로써 코드는 한층 더 이해하기 쉬워졌습니다. 추가적으로, 다른 `DaoAuthenticationProvider`가 아닌 다른 `AuthenticationProvider`로도 이해가 쉽게 확장될 수 있을 것입니다.

- 책임

객체 지향 프로그래밍(OOP, Object Oriented Programming)에서는 책임 분리와 범위 설정이 중요합니다. 이는 각 클래스나 메서드가 하나의 책임만 가지고, 담당하는 범위를 명확히 설정할 필요가 있다는 것인데요.

코드 (1)에서는 `GlboalAuthenticationConfigurerAdapter`를 이용하며 `customUserDetailsService`를 이용하지만, 실제로는 다양한 책임이 혼합되어 있습니다. 이는 `GlobalAuthenticationConfigurerAdapter`가 인증 설정을 전역적으로 적용할 때 이용되는데, 실제 코드에서는 `UserDetailsService` 구현체를 설정할 때만 이용됩니다. 즉, 현재 필요하지 않으면서도 더 많은 책임과 범위를 가지고 있습니다.

코드 (2)에서는 `DaoAuthenticationManager` 구현체 설정을 `authenticationProvider()` 메서드로 분리했고, 이 메서드는 `userDetailsService` 구현체와 `passwordEncoder` 구현체를 설정하고 반환하는 단일한 책임과 명확한 범위를 가집니다. `AuthenticationManager` 구현체를 Bean으로 올리면서도, `AuthenticationManagerBuilder`는 `authenticationProvider()` 반환값을 이용한다는 점도 명확합니다.

## 나가며

비록 코드 (2)에도 충분히 개선점은 많고, 상황에 따라 전역적으로 추가 설정이 더욱 필요한 경우에는 `GlobalAuthenticationConfigurerAdapter`를 이용해야할 수도 있겠습니다. 하지만, 현재 프로젝트 상황에서는 코드 (2)가 더욱 이해도와 책임 분리 측면에서 더 낫다고 판단했습니다.

언제든 이 글에 대해서 추가적인 의견이나 레퍼런스가 있으신 분은 환영합니다!

### Reference

https://spring.io/blog/2022/02/21/spring-security-without-the-websecurityconfigureradapter

https://dev.to/pryhmez/implementing-spring-security-6-with-spring-boot-3-a-guide-to-oauth-and-jwt-with-nimbus-for-authentication-2lhf

<br/>
