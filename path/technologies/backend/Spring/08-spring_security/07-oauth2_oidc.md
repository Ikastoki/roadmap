# OAuth2 / OIDC

OAuth2 es un estándar de autorización que permite a una aplicación acceder a recursos en nombre de un usuario sin exponer sus credenciales. OpenID Connect (OIDC) extiende OAuth2 añadiendo una capa de autenticación que proporciona información verificada sobre la identidad del usuario. Spring Security integra ambos estándares mediante módulos especializados que simplifican la configuración del cliente OAuth2, del servidor de recursos y del login social.

En un flujo OAuth2 típico, el usuario autoriza a la aplicación para acceder a sus datos en un proveedor externo (Google, GitHub, Microsoft). El proveedor emite un token de acceso que la aplicación utiliza para realizar peticiones en su nombre. OIDC añade un token de ID que contiene información verificada del usuario (email, nombre, roles) que la aplicación puede utilizar para la autenticación.

## OAuth2 Client

Un cliente OAuth2 es una aplicación que solicita acceso a recursos protegidos en nombre de un usuario. Spring Security gestiona automáticamente el flujo de redirección, el intercambio de código por tokens y la renovación de tokens.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user,user:email
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid,email,profile
        provider:
          github:
            authorization-uri: https://github.com/login/oauth/authorize
            token-uri: https://github.com/login/oauth/access_token
            user-info-uri: https://api.github.com/user
            user-name-attribute: login
```

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(oauth2 -> oauth2
            .loginPage("/login")
            .defaultSuccessUrl("/dashboard", true)
            .userInfoEndpoint(userInfo -> userInfo
                .userAuthoritiesMapper(userAuthoritiesMapper())
            )
        );

    return http.build();
}

@Bean
public GrantedAuthoritiesMapper userAuthoritiesMapper() {
    return authorities -> {
        Set<GrantedAuthority> mappedAuthorities = new HashSet<>();
        authorities.forEach(authority -> {
            if (authority instanceof OAuth2UserAuthority oauth2Auth) {
                String email = oauth2Auth.getAttributes().get("email").toString();
                // Buscar usuario en la BD y asignar roles
                mappedAuthorities.add(new SimpleGrantedAuthority("ROLE_USUARIO"));
            }
        });
        return mappedAuthorities;
    };
}
```

> Spring Boot autoconfigura los proveedores más comunes (GitHub, Google, Facebook)只需要提供 client-id y client-secret. Para otros proveedores, se deben especificar las URIs de autorización, token y información del usuario.

## OAuth2 Resource Server

Un servidor de recursos protege endpoints que consumen tokens de acceso emitidos por un servidor de autorización. Spring Security valida automáticamente los tokens JWT o de referencia contra el proveedor configurado.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.miapp.com
          jwk-set-uri: https://auth.miapp.com/.well-known/jwks.json
```

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/**").authenticated()
            .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2
            .jwt(jwt -> jwt
                .jwtAuthenticationConverter(jwtAuthenticationConverter())
            )
        );

    return http.build();
}

@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter() {
    JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
    authoritiesConverter.setAuthorityPrefix("SCOPE_");
    authoritiesConverter.setAuthoritiesClaimName("roles");

    JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
    converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
    return converter;
}
```

## OIDC Login

OIDC (OpenID Connect) extiende OAuth2 para proporcionar autenticación. Cuando se configura `oauth2Login`, Spring Security utiliza el token de ID para autenticar al usuario y extraer información del perfil verificada por el proveedor.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          mi-proveedor:
            client-id: ${CLIENT_ID}
            client-secret: ${CLIENT_SECRET}
            scope: openid,profile,email
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            client-name: Mi Proveedor
        provider:
          mi-proveedor:
            issuer-uri: https://auth.miapp.com
            authorization-uri: https://auth.miapp.com/authorize
            token-uri: https://auth.miapp.com/token
            jwk-set-uri: https://auth.miapp.com/.well-known/jwks.json
            user-info-uri: https://auth.miapp.com/userinfo
            user-name-attribute: sub
```

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/", "/error").permitAll()
            .anyRequest().authenticated()
        )
        .oauth2Login(oauth2 -> oauth2
            .loginPage("/login")
            .userInfoEndpoint(userInfo -> userInfo
                .oidcUserService(oidcUserService())
            )
        );

    return http.build();
}

@Bean
public OAuth2User oidcUserService() {
    return new OidcUser() {
        @Override
        public Map<String, Object> getClaims() {
            return Map.of();
        }

        @Override
        public OidcIdToken getIdToken() {
            return null;
        }

        @Override
        public OidcUserInfo getUserInfo() {
            return null;
        }

        @Override
        public Map<String, Object> getAttributes() {
            return Map.of();
        }

        @Override
        public Collection<? extends GrantedAuthority> getAuthorities() {
            return List.of(new SimpleGrantedAuthority("ROLE_USUARIO"));
        }

        @Override
        public String getName() {
            return "OIDC_USER";
        }
    };
}
```

> OIDC es preferable sobre OAuth2 puro para autenticación porque el token de ID está firmado y verificado, garantizando que la información del usuario no ha sido manipulada.

## Client Registration Programatica

Además de la configuración YAML, se puede registrar clientes OAuth2 de forma programática para escenarios dinámicos.

```java
@Bean
public ClientRegistrationRepository clientRegistrationRepository() {
    ClientRegistration github = ClientRegistration.withRegistrationId("github")
            .clientId("mi-client-id")
            .clientSecret("mi-client-secret")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .scope("read:user", "user:email")
            .authorizationUri("https://github.com/login/oauth/authorize")
            .tokenUri("https://github.com/login/oauth/access_token")
            .userInfoUri("https://api.github.com/user")
            .userNameAttributeName("login")
            .clientName("GitHub")
            .build();

    return new InMemoryClientRegistrationRepository(github);
}
```

La integración de OAuth2 y OIDC en Spring Security elimina la necesidad de gestionar manualmente tokens, flujos de autorización y verificación de identidad. Esto permite a las aplicaciones autenticar usuarios mediante proveedores externos de forma segura y estándar, sin implementar la complejidad de los protocolos subyacentes.

