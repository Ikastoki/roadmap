# JWT (JSON Web Tokens)

JWT es un estándar (RFC 7519) para la transmisión de información entre partes como un objeto JSON firmado digitalmente. En el contexto de la seguridad web, JWT permite implementar autenticación stateless donde el servidor no mantiene sesión del usuario: cada petición incluye el token completo con los datos de identidad y autorización, y el servidor solo verifica la firma y la expiración antes de procesar la solicitud.

Un JWT se compine de tres partes separadas por puntos: el header (algoritmo de firma y tipo de token), el payload (claims o afirmaciones como el ID del usuario, roles y fechas de expiración) y la firma (firma digital que garantiza la integridad del token). La estructura resultante es `xxxxx.yyyyy.zzzzz`, donde cada parte está codificada en Base64URL.

## Estructura de un JWT

```java
public class JwtUtil {

    private final String secretKey;
    private final long expirationMs;

    public JwtUtil(@Value("${jwt.secret}") String secretKey,
                   @Value("${jwt.expiration-ms}") long expirationMs) {
        this.secretKey = secretKey;
        this.expirationMs = expirationMs;
    }

    // Header: {"alg":"HS256","typ":"JWT"}
    // Payload: {"sub":"user@email.com","roles":["ROLE_ADMIN"],"iat":1700000000,"exp":1700003600}
    // Firma: HMAC-SHA256(base64(header) + "." + base64(payload), secretKey)

    public String generarToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()));

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(userDetails.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expirationMs))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }

    private Key getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public boolean isTokenValid(String token, UserDetails userDetails) {
        String username = extractUsername(token);
        return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
    }

    private boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }
}
```

> La clave secreta utilizada para firmar JWTs debe almacenarse de forma segura (variables de entorno, vault, etc.) y nunca debe hardcodearse en el código fuente. Una clave comprometida permite a un atacante generar tokens válidos.

## JWT Filter

El filtro JWT intercepta cada petición, extrae el token del header `Authorization: Bearer <token>`, lo valida y establece la autenticación en el `SecurityContext`.

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtUtil jwtUtil, UserDetailsService userDetailsService) {
        this.jwtUtil = jwtUtil;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                   HttpServletResponse response,
                                   FilterChain filterChain) throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);

        try {
            String username = jwtUtil.extractUsername(token);

            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                if (jwtUtil.isTokenValid(token, userDetails)) {
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,
                            userDetails.getAuthorities()
                        );
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);
                }
            }
        } catch (Exception e) {
            logger.error("Error al procesar JWT: " + e.getMessage());
        }

        filterChain.doFilter(request, response);
    }
}
```

```java
@Configuration
@EnableWebSecurity
public class JwtSecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;

    public JwtSecurityConfig(JwtAuthenticationFilter jwtAuthFilter,
                            UserDetailsService userDetailsService) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

## Token generation y validation

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;

    public AuthController(AuthenticationManager authenticationManager,
                         JwtUtil jwtUtil,
                         UserDetailsService userDetailsService) {
        this.authenticationManager = authenticationManager;
        this.jwtUtil = jwtUtil;
        this.userDetailsService = userDetailsService;
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody AuthRequest request) {
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.email(), request.password())
        );

        UserDetails userDetails = userDetailsService.loadUserByUsername(request.email());
        String token = jwtUtil.generarToken(userDetails);

        return ResponseEntity.ok(new AuthResponse(token, "Bearer"));
    }

    @PostMapping("/validar")
    public ResponseEntity<Map<String, Object>> validarToken(
            @RequestHeader("Authorization") String authHeader) {

        String token = authHeader.substring(7);
        String username = jwtUtil.extractUsername(token);
        boolean valido = jwtUtil.isTokenValid(token, userDetailsService.loadUserByUsername(username));

        return ResponseEntity.ok(Map.of(
            "valido", valido,
            "usuario", username
        ));
    }
}

public record AuthRequest(String email, String password) {}
public record AuthResponse(String token, String tipo) {}
```

## Refresh Tokens

Los refresh tokens permiten obtener nuevos tokens de acceso sin que el usuario tenga que volver a autenticarse. Cuando el access token expira, el cliente envía el refresh token para recibir un par nuevo de tokens.

```java
@Service
public class TokenService {

    private final JwtUtil jwtUtil;
    private final RefreshTokenRepository refreshTokenRepository;
    private final UserDetailsService userDetailsService;

    public TokenService(JwtUtil jwtUtil,
                       RefreshTokenRepository refreshTokenRepository,
                       UserDetailsService userDetailsService) {
        this.jwtUtil = jwtUtil;
        this.refreshTokenRepository = refreshTokenRepository;
        this.userDetailsService = userDetailsService;
    }

    public AuthResponse generarTokens(String email) {
        UserDetails userDetails = userDetailsService.loadUserByUsername(email);
        String accessToken = jwtUtil.generarToken(userDetails);
        String refreshToken = generarRefreshToken(email);

        return new AuthResponse(accessToken, refreshToken, "Bearer");
    }

    public AuthResponse refrescarToken(String refreshToken) {
        RefreshToken token = refreshTokenRepository.findByToken(refreshToken)
                .orElseThrow(() -> new IllegalArgumentException("Refresh token invalido"));

        if (token.isExpirado()) {
            throw new IllegalArgumentException("Refresh token expirado");
        }

        UserDetails userDetails = userDetailsService.loadUserByUsername(token.getEmail());
        String nuevoAccessToken = jwtUtil.generarToken(userDetails);
        String nuevoRefreshToken = generarRefreshToken(token.getEmail());

        refreshTokenRepository.delete(token);

        return new AuthResponse(nuevoAccessToken, nuevoRefreshToken, "Bearer");
    }

    private String generarRefreshToken(String email) {
        String token = UUID.randomUUID().toString();
        RefreshToken refreshToken = new RefreshToken(token, email, Instant.now().plus(Duration.ofDays(7)));
        refreshTokenRepository.save(refreshToken);
        return token;
    }
}

@Entity
@Table(name = "refresh_tokens")
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String token;

    @Column(nullable = false)
    private String email;

    @Column(nullable = false)
    private Instant expiracion;

    public RefreshToken() {}

    public RefreshToken(String token, String email, Instant expiracion) {
        this.token = token;
        this.email = email;
        this.expiracion = expiracion;
    }

    public String getToken() { return token; }
    public String getEmail() { return email; }
    public boolean isExpirado() { return Instant.now().isAfter(expiracion); }
}

public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
    Optional<RefreshToken> findByToken(String token);
}
```

> Los refresh tokens deben almacenarse de forma segura en la base de datos y deben ser invalidate cuando se usan (rotate). Un refresh token solo debe usarse una vez para evitar que un token comprometido siga siendo válido indefinidamente.

La autenticación stateless con JWT elimina la dependencia de sesiones server-side, lo que facilita la escalabilidad horizontal. Cada instancia del servidor puede validar tokens de forma independiente, sin necesidad de compartir estado entre nodos. Sin embargo, esto implica que la revocación de tokens solo puede lograrse mediante expiración o黑名单 en la base de datos.

