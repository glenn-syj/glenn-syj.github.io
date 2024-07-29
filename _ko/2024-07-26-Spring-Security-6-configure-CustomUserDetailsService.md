---
title: Spring Security 6에서 CustomUserDetailsService 이용하기
lang: ko
layout: post
---

## 들어가며
### Disclaimer
- Last Update: 2024/07/29

- SSAFY 내 이전 프로젝트에서 WebSocket을 맡았으나, 인원 부족으로 인한 팀 해체로 새로운 팀에서 Spring Security를 이용한 기존 인증과 인가 코드를 수정하고 개선하는 일에 우선적으로 돌입했습니다.
- 이전 코드는 컨트롤러 기반의 로그인이 아니라, Fitler 위주로 JWT Filter 및 Login Filter를 이용하고 있었습니다.
- Spring Security를 처음 이용하며, 마주하는 전체적인 그림이나 트러블슈팅에 대해 자주 글을 쓸 예정입니다.

<br/>

## 문제 상황

### CustomUserService에 대해서

`CustomUserDetailsService`는 Spring Security에서 제공하는 `UserDetailsService` 인터페이스의 구현체입니다. 이름을 하나씩 뜯어보자면, Custom + UserDetails + Service라고 볼 수 있는데요. Service가 서비스 레이어를 의미한다면, Custom은 요구에 맞게 변경했다는 뜻이겠습니다. 여기에서 중요한 것은 `UserDetails`입니다. 

Spring Security에서 `UserDetails` 인터페이스는 권한과 관련된 `getAuthorities()`, 비밀번호와 관련된 `getPassword()`, 그리고 유저의 구분자가 되는 유저네임과 관련된 `getUsername()` 메서드가 있습니다. 물론, 해당 유저의 상태를 나타내는 디폴트 메서드들 역시 오버라이드 해야합니다. 그런데 제가 앞서 "유저의 구분자"라는 표현을 썼었죠? 실제로 `UserDetails` 인터페이스를 구현한 Spring Security 내 `User` 클래스는 다음과 같이 username과 관련된 코드를 구성하고 있습니다.

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



### Reference


<br/>
