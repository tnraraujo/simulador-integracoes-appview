# Matriz RBAC - Controle de Acesso
**App View v2.0 - Segurança e Autorização**

**Versão:** 2.0.0  
**Data:** Janeiro de 2026  

---

## Visão Geral

Sistema de controle de acesso baseado em **RBAC (Role-Based Access Control)** com 5 perfis padrão e permissões granulares por recurso.

### Princípios
- **Privilégio Mínimo:** Usuários recebem apenas permissões necessárias
- **Segregação de Funções:** Separação clara entre perfis operacionais e administrativos
- **Auditoria Completa:** Todos os acessos são registrados
- **Mapeamento Azure AD:** Grupos do Azure AD são automaticamente mapeados para perfis

---

## Perfis do Sistema

| Perfil | Descrição | Público-Alvo | Total Permissões |
|--------|-----------|--------------|------------------|
| **ADMIN** | Administrador do sistema | TI, Suporte | 47 (todas) |
| **OPERADOR** | Operador de sinistros e clientes | Operações, BackOffice | 28 |
| **JURIDICO** | Acesso ao Portal PJ | Departamento Jurídico | 12 |
| **GESTOR** | Visualização de relatórios e dashboards | Gerência, Liderança | 15 |
| **CONSULTA** | Apenas leitura (readonly) | Auditoria, Compliance | 8 |

---

## Matriz de Permissões por Recurso

### Módulo Sinistro

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `sinistro:create` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `sinistro:read` | ✅ | ✅ | ❌ | ✅ | ✅ |
| `sinistro:update` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `sinistro:delete` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `sinistro:reenviar_ocr` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `sinistro:forcar_roteamento` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `sinistro:baixar_documento` | ✅ | ✅ | ❌ | ❌ | ❌ |

### Módulo Cliente

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `cliente:create` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `cliente:read` | ✅ | ✅ | ❌ | ✅ | ✅ |
| `cliente:update` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `cliente:delete` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `cliente:baixar_documento` | ✅ | ✅ | ❌ | ❌ | ❌ |

### Módulo Contrato (BOD)

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `contrato:create` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `contrato:read` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `contrato:update` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `contrato:delete` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `contrato:renovar` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `contrato:cancelar` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `contrato:baixar_proposta` | ✅ | ✅ | ✅ | ✅ | ❌ |

### Módulo Comunicação

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `comunicacao:create` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `comunicacao:read` | ✅ | ✅ | ❌ | ✅ | ✅ |
| `comunicacao:reenviar` | ✅ | ✅ | ❌ | ❌ | ❌ |

### Módulo Perícia

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `pericia:create` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `pericia:read` | ✅ | ✅ | ❌ | ✅ | ✅ |
| `pericia:update` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `pericia:cancelar` | ✅ | ✅ | ❌ | ❌ | ❌ |

### Módulo Portal PJ

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `portal:acessar` | ✅ | ❌ | ✅ | ❌ | ❌ |
| `portal:consultar_propostas` | ✅ | ❌ | ✅ | ❌ | ❌ |
| `portal:baixar_documentos` | ✅ | ❌ | ✅ | ❌ | ❌ |
| `portal:consultar_portabilidades` | ✅ | ❌ | ✅ | ❌ | ❌ |

### Módulo IAM

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `usuario:create` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `usuario:read` | ✅ | ❌ | ❌ | ✅ | ❌ |
| `usuario:update` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `usuario:delete` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `usuario:resetar_senha` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `perfil:create` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `perfil:read` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `perfil:update` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `perfil:delete` | ✅ | ❌ | ❌ | ❌ | ❌ |

### Módulo Documento

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `documento:create` | ✅ | ✅ | ✅ | ❌ | ❌ |
| `documento:read` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `documento:update` | ✅ | ✅ | ❌ | ❌ | ❌ |
| `documento:delete` | ✅ | ❌ | ❌ | ❌ | ❌ |
| `documento:gerar_link_temp` | ✅ | ✅ | ✅ | ❌ | ❌ |

### Módulo Relatórios

| Permissão | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|-----------|-------|----------|----------|--------|----------|
| `relatorio:read` | ✅ | ❌ | ❌ | ✅ | ✅ |
| `relatorio:dashboard` | ✅ | ❌ | ❌ | ✅ | ❌ |
| `relatorio:exportar` | ✅ | ❌ | ❌ | ✅ | ❌ |
| `relatorio:auditoria` | ✅ | ❌ | ❌ | ❌ | ❌ |

---

## Mapeamento Azure AD → Perfis

| Grupo Azure AD | Perfil App View | Descrição |
|----------------|-----------------|-----------|
| `AppView-Admins` | ADMIN | Administradores do sistema |
| `AppView-Operadores` | OPERADOR | Operadores de sinistros e clientes |
| `AppView-Juridico` | JURIDICO | Departamento jurídico (Portal PJ) |
| `AppView-Gestores` | GESTOR | Gerência e liderança |
| `AppView-Auditores` | CONSULTA | Auditoria e compliance |

### Configuração no Azure AD

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://login.microsoftonline.com/{tenant-id}/v2.0
          
app:
  security:
    role-mapping:
      AppView-Admins: ADMIN
      AppView-Operadores: OPERADOR
      AppView-Juridico: JURIDICO
      AppView-Gestores: GESTOR
      AppView-Auditores: CONSULTA
```

---

## Implementação Spring Security

### 1. Entidade Permissão

```java
@Entity
@Table(name = "tb_permissao", schema = "iam")
public class Permissao {
    
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;
    
    @Column(nullable = false, unique = true, length = 100)
    private String codigo; // Ex: "sinistro:create"
    
    @Column(nullable = false, length = 200)
    private String descricao;
    
    @Column(nullable = false, length = 50)
    private String modulo; // Ex: "SINISTRO"
    
    @Column(nullable = false, length = 50)
    private String acao; // Ex: "CREATE"
    
    @Column(nullable = false)
    private Boolean ativo = true;
}
```

### 2. Anotação de Autorização

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority(#permission)")
public @interface RequiresPermission {
    String value();
}
```

### 3. Uso em Controllers

```java
@RestController
@RequestMapping("/api/sinistros")
public class SinistroController {
    
    @PostMapping
    @RequiresPermission("sinistro:create")
    public ResponseEntity<SinistroResponse> receberSinistro(
            @RequestBody @Valid SinistroRequest request,
            @RequestHeader("Idempotency-Key") String idempotencyKey) {
        // ...
    }
    
    @GetMapping("/{id}")
    @RequiresPermission("sinistro:read")
    public ResponseEntity<SinistroResponse> consultar(@PathVariable UUID id) {
        // ...
    }
    
    @DeleteMapping("/{id}")
    @RequiresPermission("sinistro:delete")
    public ResponseEntity<Void> deletar(@PathVariable UUID id) {
        // ...
    }
}
```

### 4. Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/api/jarvis/callback").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("");
        
        JwtAuthenticationConverter jwtAuthenticationConverter = 
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter);
        
        return jwtAuthenticationConverter;
    }
}
```

---

## Controle de Acesso em Nível de Dados (Row-Level Security)

### Regras
1. **OPERADOR:** Visualiza apenas sinistros dos últimos 6 meses
2. **JURIDICO:** Acessa apenas contratos corporativos (Portal PJ)
3. **GESTOR:** Visualiza todos os dados, sem edição
4. **CONSULTA:** Dados sensíveis são mascarados (CPF parcial)

### Implementação com Specifications

```java
@Component
public class SinistroSecuritySpecification {
    
    public static Specification<Sinistro> aplicarFiltrosSeguranca(
            Authentication authentication) {
        
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            
            Set<String> roles = extractRoles(authentication);
            
            // OPERADOR vê apenas últimos 6 meses
            if (roles.contains("OPERADOR")) {
                LocalDate seisMesesAtras = LocalDate.now().minusMonths(6);
                predicates.add(cb.greaterThanOrEqualTo(
                    root.get("dataRecebimento"), seisMesesAtras));
            }
            
            // GESTOR e CONSULTA não têm filtros adicionais
            
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
    
    private static Set<String> extractRoles(Authentication auth) {
        return auth.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toSet());
    }
}
```

### Uso no Repository

```java
public interface SinistroRepository extends JpaRepository<Sinistro, UUID>,
        JpaSpecificationExecutor<Sinistro> {
}
```

### Uso no Use Case

```java
public class ConsultarSinistroUseCase {
    
    public Page<SinistroDTO> executar(
            SinistroFiltro filtro,
            Pageable pageable,
            Authentication authentication) {
        
        Specification<Sinistro> spec = Specification.where(
            SinistroSpecifications.comFiltros(filtro))
            .and(SinistroSecuritySpecification.aplicarFiltrosSeguranca(authentication));
        
        return repository.findAll(spec, pageable)
            .map(SinistroMapper::toDTO);
    }
}
```

---

## Auditoria de Acessos

### Tabela de Audit Log

```sql
CREATE TABLE iam.tb_log_acesso (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    usuario_id UUID NOT NULL REFERENCES iam.tb_usuario(id),
    email VARCHAR(255) NOT NULL,
    perfil VARCHAR(50) NOT NULL,
    permissao VARCHAR(100) NOT NULL,
    recurso VARCHAR(255) NOT NULL,
    acao VARCHAR(50) NOT NULL,
    sucesso BOOLEAN NOT NULL,
    ip_address VARCHAR(45),
    user_agent VARCHAR(500),
    request_id VARCHAR(36),
    data_hora TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_log_acesso_usuario ON iam.tb_log_acesso(usuario_id);
CREATE INDEX idx_log_acesso_data ON iam.tb_log_acesso(data_hora DESC);
CREATE INDEX idx_log_acesso_permissao ON iam.tb_log_acesso(permissao);
```

### Interceptor de Auditoria

```java
@Aspect
@Component
public class AccessAuditAspect {
    
    private final LogAcessoRepository logRepository;
    
    @Around("@annotation(requiresPermission)")
    public Object auditarAcesso(
            ProceedingJoinPoint joinPoint,
            RequiresPermission requiresPermission) throws Throwable {
        
        Authentication auth = SecurityContextHolder.getContext()
            .getAuthentication();
        HttpServletRequest request = getCurrentRequest();
        
        String permissao = requiresPermission.value();
        boolean sucesso = false;
        
        try {
            Object result = joinPoint.proceed();
            sucesso = true;
            return result;
        } finally {
            LogAcesso log = LogAcesso.builder()
                .email(auth.getName())
                .perfil(extractPerfil(auth))
                .permissao(permissao)
                .recurso(request.getRequestURI())
                .acao(request.getMethod())
                .sucesso(sucesso)
                .ipAddress(getClientIp(request))
                .userAgent(request.getHeader("User-Agent"))
                .requestId(MDC.get("requestId"))
                .dataHora(Instant.now())
                .build();
            
            logRepository.save(log);
        }
    }
}
```

---

## Cenários de Teste RBAC

### Testes de Autorização

```java
@SpringBootTest
@AutoConfigureMockMvc
class SinistroControllerAuthorizationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    @WithMockUser(authorities = {"OPERADOR"})
    void operadorPodeCriarSinistro() throws Exception {
        mockMvc.perform(post("/api/sinistros")
                .contentType(APPLICATION_JSON)
                .content(sinistroJson))
            .andExpect(status().isAccepted());
    }
    
    @Test
    @WithMockUser(authorities = {"CONSULTA"})
    void consultaNaoPodeCriarSinistro() throws Exception {
        mockMvc.perform(post("/api/sinistros")
                .contentType(APPLICATION_JSON)
                .content(sinistroJson))
            .andExpect(status().isForbidden());
    }
    
    @Test
    @WithMockUser(authorities = {"JURIDICO"})
    void juridicoNaoAcessaSinistros() throws Exception {
        mockMvc.perform(get("/api/sinistros/123"))
            .andExpect(status().isForbidden());
    }
    
    @Test
    @WithMockUser(authorities = {"JURIDICO"})
    void juridicoAcessaPortalPJ() throws Exception {
        mockMvc.perform(get("/api/portal/propostas"))
            .andExpect(status().isOk());
    }
    
    @Test
    @WithMockUser(authorities = {"ADMIN"})
    void adminAcessaTudo() throws Exception {
        mockMvc.perform(delete("/api/sinistros/123"))
            .andExpect(status().isNoContent());
    }
}
```

---

## Permissões Detalhadas por Perfil

### ADMIN (47 permissões)

**Sinistro:** create, read, update, delete, reenviar_ocr, forcar_roteamento, baixar_documento  
**Cliente:** create, read, update, delete, baixar_documento  
**Contrato:** create, read, update, delete, renovar, cancelar, baixar_proposta  
**Comunicação:** create, read, reenviar  
**Perícia:** create, read, update, cancelar  
**Portal:** acessar, consultar_propostas, baixar_documentos, consultar_portabilidades  
**IAM:** usuario:*, perfil:*  
**Documento:** create, read, update, delete, gerar_link_temp  
**Relatórios:** read, dashboard, exportar, auditoria  

### OPERADOR (28 permissões)

**Sinistro:** create, read, update, reenviar_ocr, forcar_roteamento, baixar_documento  
**Cliente:** create, read, update, baixar_documento  
**Contrato:** create, read, update, renovar, cancelar, baixar_proposta  
**Comunicação:** create, read, reenviar  
**Perícia:** create, read, update, cancelar  
**Documento:** create, read, update, gerar_link_temp  

### JURIDICO (12 permissões)

**Contrato:** create, read, update, renovar, baixar_proposta  
**Portal:** acessar, consultar_propostas, baixar_documentos, consultar_portabilidades  
**Documento:** create, read, gerar_link_temp  

### GESTOR (15 permissões)

**Sinistro:** read  
**Cliente:** read  
**Contrato:** read, baixar_proposta  
**Comunicação:** read  
**Perícia:** read  
**Documento:** read  
**Relatórios:** read, dashboard, exportar  
**IAM:** usuario:read  

### CONSULTA (8 permissões)

**Sinistro:** read  
**Cliente:** read  
**Contrato:** read  
**Comunicação:** read  
**Perícia:** read  
**Documento:** read  
**Relatórios:** read  

---

## Matriz Resumida (Visão por Módulo)

| Módulo | ADMIN | OPERADOR | JURIDICO | GESTOR | CONSULTA |
|--------|-------|----------|----------|--------|----------|
| Sinistro | 🔓 Full | 🔓 CRUD | 🔒 Nenhum | 👁️ Read | 👁️ Read |
| Cliente | 🔓 Full | 🔓 CRUD | 🔒 Nenhum | 👁️ Read | 👁️ Read |
| Contrato | 🔓 Full | 🔓 CRUD | 🔓 CRUD | 👁️ Read | 👁️ Read |
| Comunicação | 🔓 Full | 🔓 CRUD | 🔒 Nenhum | 👁️ Read | 👁️ Read |
| Perícia | 🔓 Full | 🔓 CRUD | 🔒 Nenhum | 👁️ Read | 👁️ Read |
| Portal PJ | 🔓 Full | 🔒 Nenhum | 🔓 Full | 🔒 Nenhum | 🔒 Nenhum |
| IAM | 🔓 Full | 🔒 Nenhum | 🔒 Nenhum | 👁️ Read | 🔒 Nenhum |
| Documento | 🔓 Full | 🔓 CRUD | 🔓 CRU | 👁️ Read | 👁️ Read |
| Relatórios | 🔓 Full | 🔒 Nenhum | 🔒 Nenhum | 🔓 Full | 👁️ Read |

**Legenda:**  
🔓 = Acesso completo  
👁️ = Apenas leitura  
🔒 = Nenhum acesso  

---

## Boas Práticas de Segurança

1. **Princípio do Menor Privilégio:** Sempre atribuir o perfil mais restritivo possível
2. **Revisão Periódica:** Revisar acessos trimestralmente
3. **Offboarding:** Desativar usuários imediatamente após desligamento
4. **Auditoria Contínua:** Monitorar logs de acesso anormais (ex: acessos fora do horário comercial)
5. **MFA Obrigatório:** Azure AD configurado com autenticação multifator
6. **Rotação de Tokens:** Refresh tokens com validade máxima de 30 dias
7. **Segregação:** Ambientes DEV/UAT/PRD com controles de acesso separados

---

**Documento elaborado por:** Equipe de Segurança e Arquitetura  
**Última atualização:** Janeiro de 2026
