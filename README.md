Experiment 8: React Frontend + Spring Boot Backend Integration Overview This project demonstrates a complete full-stack integration between a React frontend and a Spring Boot backend, implementing secure API communication, CORS handling, JWT-based authentication, and proper error handling with response codes.

Course Competencies: CO2 (Build web applications using frameworks), CO4 (Implement secure authentication mechanisms)

Project Architecture Backend: Spring Boot (Java 17) REST API with public and protected endpoints JWT-based authentication & authorization Global CORS configuration Spring Security integration Form validation with proper HTTP response codes Frontend: React (Vite) Axios for HTTP requests with JWT token management Fetch API for alternative public API calls React Router for navigation User-friendly forms with error/success feedback Automatic redirect to login on unauthorized access Requirements Implementation Requirement A: Public GET API with Table Display Backend Implementation Public API Endpoint: GET /api/public/posts

// controller/PublicController.java @RestController @RequestMapping("/api/public") public class PublicController { @GetMapping("/posts") public List getPosts() { return publicApiService.fetchPosts(); } } Fetches posts from a public external API (JSONPlaceholder) Returns data directly to the frontend No authentication required Automatically included in CORS whitelist Frontend Implementation (2 Approaches) Approach 1: Using Axios (components/PublicPostsTable.jsx)

import api from '../api';

useEffect(() => { api.get('/api/public/posts') .then((response) => setPosts(response.data.slice(0, 10))) .catch(() => setError('Failed to load posts from backend')); }, []); Approach 2: Using Fetch API (components/PublicPostsTableFetch.jsx)

fetch('http://localhost:8080/api/public/posts') .then((response) => response.json()) .then((data) => setPosts(data.slice(0, 10))) .catch((err) => setError(Fetch failed: ${err.message})); Features:

Displays posts in a formatted table with ID, User ID, Title, and Body Error handling with user-friendly messages No JWT token required Demonstrates both Axios and Fetch approaches Requirement B: Form Submission with Response Code Handling Backend Implementation Registration Endpoint: POST /api/auth/register

@PostMapping("/register") public ResponseEntity register(@Valid @RequestBody RegisterRequest request) { boolean created = userService.register(request.getUsername(), request.getPassword());

if (!created) {
    return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new AuthResponse(null, "Username already exists")); // 409
}

return ResponseEntity.status(HttpStatus.CREATED)
        .body(new AuthResponse(null, "Registration successful")); // 201
} Login Endpoint: POST /api/auth/login

@PostMapping("/login") public ResponseEntity login(@Valid @RequestBody LoginRequest request) { boolean valid = userService.validate(request.getUsername(), request.getPassword());

if (!valid) {
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new AuthResponse(null, "Invalid username or password")); // 401
}

String token = jwtService.generateToken(request.getUsername());
return ResponseEntity.ok(new AuthResponse(token, "Login successful")); // 200
} Product Creation Endpoint: POST /api/products (Protected)

@PostMapping public ResponseEntity<?> createProduct(@Valid @RequestBody ProductRequest request) { return ResponseEntity.status(HttpStatus.CREATED) .body(Map.of( "message", "Product created successfully", "productName", request.getName(), "price", request.getPrice() )); // 201 } Response Codes Used Status Code Scenario Example 200 Success Login returns token 201 Created Registration, Product creation 400 Validation Error Invalid input 401 Unauthorized Invalid credentials, missing/expired token 409 Conflict User already exists Frontend Implementation Registration Component (components/RegisterForm.jsx)

try { const response = await api.post('/api/auth/register', form); setMessage(response.data.message); } catch (error) { if (error.response?.status === 409) { setMessage(error.response.data.message); // "Username already exists" } else if (error.response?.status === 400) { setMessage('Validation failed'); } else { setMessage('Registration failed'); } } Login Component (components/Login.jsx)

try { const response = await api.post('/api/auth/login', form); localStorage.setItem('token', response.data.token); setMessage('Login successful'); navigate('/products'); } catch (error) { if (error.response?.status === 401) { setMessage('Invalid username or password'); } else { setMessage('Login failed'); } } Product Form Component (components/ProductForm.jsx)

try { const response = await api.post('/api/products', {...}); setMessage(response.data.message); // Success } catch (error) { if (error.response?.status === 401) { setMessage('Unauthorized. Redirecting to login...'); } else if (error.response?.status === 400) { setMessage('Validation failed'); } } Features:

Status code-based error messages User-friendly success confirmations Form validation feedback Color-coded UI (green for success, red for errors) Requirement C: CORS Configuration & JWT Protected APIs Backend: Global CORS Configuration CorsConfig.java

@Configuration public class CorsConfig { @Bean public CorsConfigurationSource corsConfigurationSource() { CorsConfiguration configuration = new CorsConfiguration(); configuration.setAllowedOrigins(List.of("http://localhost:3000")); configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS")); configuration.setAllowedHeaders(List.of("*")); configuration.setAllowCredentials(true);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", configuration);
    return source;
}
} Configuration Details:

Allowed Origin: http://localhost:3000 (React dev server) Allowed Methods: GET, POST, PUT, DELETE, OPTIONS Allowed Headers: All (*) Credentials: Allowed for token-based auth Backend: Spring Security Configuration SecurityConfig.java

@Configuration public class SecurityConfig { @Bean public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception { http .csrf(csrf -> csrf.disable()) .cors(Customizer.withDefaults()) .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) .authorizeHttpRequests(auth -> auth .requestMatchers(HttpMethod.OPTIONS, "/").permitAll() .requestMatchers("/api/public/", "/api/auth/**").permitAll() .anyRequest().authenticated() // All other endpoints require authentication ) .exceptionHandling(ex -> ex.authenticationEntryPoint( (request, response, authException) -> response.sendError(HttpServletResponse.SC_UNAUTHORIZED, "Unauthorized") )) .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
} Security Rules:

Public endpoints: /api/public/** and /api/auth/** Protected endpoints: All others (require valid JWT) CORS enabled for specified origin CSRF disabled (stateless API) Stateless session management Backend: JWT Authentication Filter JwtAuthFilter.java

@Component public class JwtAuthFilter extends OncePerRequestFilter { @Override protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException { String authHeader = request.getHeader("Authorization");

    if (authHeader != null && authHeader.startsWith("Bearer ")) {
        String token = authHeader.substring(7);
        try {
            String username = jwtService.extractUsername(token);
            if (jwtService.isTokenValid(token, username)) {
                UsernamePasswordAuthenticationToken authentication =
                    new UsernamePasswordAuthenticationToken(username, null, AuthorityUtils.NO_AUTHORITIES);
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (Exception ignored) {
        }
    }
    
    filterChain.doFilter(request, response);
}
} Filter Logic:

Extracts JWT token from Authorization: Bearer header Validates token expiration and signature Sets authenticated user in security context Allows filter chain to continue (no exception on invalid token) Backend: JWT Service JwtService.java

@Service public class JwtService { @Value("${jwt.secret}") private String secret;

public String generateToken(String username) {
    return Jwts.builder()
        .subject(username)
        .issuedAt(new Date())
        .expiration(new Date(System.currentTimeMillis() + 60 * 60 * 1000)) // 1 hour
        .signWith(getSignKey())
        .compact();
}

public boolean isTokenValid(String token, String username) {
    return username != null
        && username.equals(extractUsername(token))
        && extractAllClaims(token).getExpiration().after(new Date());
}
} JWT Details:

Algorithm: HS256 (HMAC with SHA-256) Expiration: 1 hour Secret: Configured in application.properties (default: MyVerySecretKeyMyVerySecretKey12345) Contains: Username as subject, issued-at, expiration Frontend: Axios Interceptor for JWT Management api.js

import axios from 'axios';

const api = axios.create({ baseURL: 'http://localhost:8080' });

// Request Interceptor: Attach token to every request api.interceptors.request.use( (config) => { const token = localStorage.getItem('token'); if (token) { config.headers.Authorization = Bearer ${token}; } return config; }, (error) => Promise.reject(error) );

// Response Interceptor: Handle 401 responses api.interceptors.response.use( (response) => response, (error) => { if (error.response && error.response.status === 401) { localStorage.removeItem('token'); window.location.href = '/login'; // Redirect to login } return Promise.reject(error); } );

export default api; Interceptor Logic:

Request: Automatically attaches JWT token from localStorage to Authorization header Response: If 401 response received, clear token and redirect to login page Token Storage: Token stored in browser's localStorage after login Frontend: Login & Token Storage Login.jsx

const handleSubmit = async (e) => { try { const response = await api.post('/api/auth/login', form); localStorage.setItem('token', response.data.token); // Store token navigate('/products'); } catch (error) { // Handle error } }; Frontend: Protected API Call ProductForm.jsx

const handleSubmit = async (e) => { try { const response = await api.post('/api/products', { name: form.name, price: Number(form.price) }); // Token automatically attached by interceptor setMessage(response.data.message); } catch (error) { if (error.response?.status === 401) { // Auto-redirected to login by response interceptor setMessage('Unauthorized. Redirecting to login...'); } } }; Setup & Installation Backend Setup Navigate to backend directory:

cd backend Build and run with Maven:

mvn clean install mvn spring-boot:run Backend runs on: http://localhost:8080

Configure JWT secret (optional, in application.properties):

jwt.secret=MyVerySecretKeyMyVerySecretKey12345 server.port=8080 Frontend Setup Navigate to frontend directory:

cd frontend Install dependencies:

npm install Run development server:

npm run dev Frontend runs on: http://localhost:3000

API Endpoints Summary Public Endpoints (No Authentication Required) Method Endpoint Description GET /api/public/posts Fetch public posts from external API POST /api/auth/register Register new user POST /api/auth/login Login and get JWT token Protected Endpoints (JWT Required) Method Endpoint Description Headers Required POST /api/products Create product Authorization: Bearer Application Flow Registration & Login Flow

User enters username/password in Register Form

Frontend sends POST /api/auth/register

Backend validates and creates user

Backend returns 201 (success) or 409 (user exists)

Frontend displays appropriate message

User enters credentials in Login Form

Frontend sends POST /api/auth/login

Backend validates credentials

Backend returns JWT token (200) or error (401)

Frontend stores token in localStorage

Frontend redirects to protected route Protected API Call Flow

User clicks "Create Product"

Frontend sends POST /api/products with token

Axios interceptor automatically adds: Authorization: Bearer

Backend receives request

JwtAuthFilter validates token and sets authentication context

SecurityConfig checks if user is authenticated

ProductController processes request

Backend returns 201 with product data

Frontend displays success message

If token is expired or invalid:

Backend returns 401 Unauthorized
Response interceptor catches 401
Token is cleared from localStorage
User is redirected to /login
Frontend displays "Unauthorized. Redirecting to login..." Technology Stack Backend Framework: Spring Boot 3.3.5 Language: Java 17 Security: Spring Security + JWT (JJWT) Build: Maven Database: In-memory (Users in UserService) Frontend Framework: React 18.3.1 Build Tool: Vite 5.4.10 Routing: React Router 6.30.1 HTTP Client: Axios 1.9.0 CSS: Vanilla CSS with responsive design Key Features Implemented ✅ CORS Handling
Global configuration in CorsConfig.java Whitelist specific origin (localhost:3000) Allow credentials for token-based authentication ✅ JWT Authentication

Token generation on login Token validation on protected requests 1-hour expiration HS256 algorithm ✅ Response Code Handling

200: OK (login success) 201: Created (registration, product creation) 400: Bad Request (validation errors) 401: Unauthorized (invalid token/credentials) 409: Conflict (duplicate username) ✅ Frontend Error Handling

Status code-based messages Automatic redirect on 401 User-friendly error displays Color-coded responses (green/red) ✅ Public & Protected Endpoints

Public endpoints for registration, login, and data fetching Protected endpoints require valid JWT token ✅ Both Fetch & Axios

Fetch API example: PublicPostsTableFetch.jsx Axios with interceptors: All other components Security Considerations Secret Key: Keep JWT secret in environment variables (not hardcoded) HTTPS: Use HTTPS in production (not just HTTP) Token Expiration: Currently 1 hour (configurable) CORS Origin: Restrict to specific trusted origins CSRF: Disabled because API is stateless (acceptable for token auth) Password Security: Hash passwords in UserService (use BCrypt) Common Issues & Solutions Issue Solution CORS error when calling backend Check CorsConfig.java has correct origin URL 401 Unauthorized on protected route Ensure token is stored in localStorage after login Token not sent with request Verify Axios interceptor in api.js is configured correctly Stuck on login redirect loop Clear localStorage, restart browser, re-login Backend not found (404) Ensure backend is running on http://localhost:8080 File Structure experiment8-react-spring-boot/ ├── backend/ │ ├── pom.xml │ └── src/main/java/com/example/demo/ │ ├── DemoApplication.java │ ├── config/ │ │ ├── CorsConfig.java (Global CORS configuration) │ │ └── SecurityConfig.java (Spring Security + JWT setup) │ ├── controller/ │ │ ├── AuthController.java (Register/Login endpoints) │ │ ├── PublicController.java (Public API endpoints) │ │ └── ProductController.java (Protected API endpoints) │ ├── dto/ │ │ ├── AuthResponse.java │ │ ├── LoginRequest.java │ │ ├── PostDto.java │ │ ├── ProductRequest.java │ │ └── RegisterRequest.java │ ├── security/ │ │ ├── JwtAuthFilter.java (JWT filter) │ │ └── JwtService.java (JWT token generation/validation) │ └── service/ │ ├── PublicApiService.java │ └── UserService.java └── frontend/ ├── package.json ├── vite.config.js ├── index.html └── src/ ├── api.js (Axios instance with interceptors) ├── App.jsx (Main component + routing) ├── main.jsx ├── styles.css └── components/ ├── Login.jsx (JWT login form) ├── RegisterForm.jsx (User registration) ├── PublicPostsTable.jsx (GET with Axios) ├── PublicPostsTableFetch.jsx (GET with Fetch API) └── ProductForm.jsx (Protected POST request) Course Competencies Covered CO2: Build web applications using frameworks ✅ React framework for frontend ✅ Spring Boot framework for backend ✅ REST API design and implementation ✅ Component-based architecture ✅ Routing and navigation CO4: Implement secure authentication mechanisms ✅ User registration and login ✅ JWT token generation and validation ✅ Protected API endpoints ✅ Token-based authorization ✅ Automatic 401 handling and redirect ✅ Secure password handling Testing the Application Test Registration Navigate to /register Enter username and password Click "Register" Expected: "Registration successful" message Try registering with same username Expected: "Username already exists" message Test Login Navigate to /login Enter registered credentials Click "Login" Expected: Redirected to /products page Try with invalid credentials Expected: "Invalid username or password" message Test Public APIs Navigate to / (Axios approach) or /fetch-posts (Fetch approach) Expected: Table displays 10 posts from external API No authentication required Test Protected API Must login first to get JWT token Navigate to /products Enter product name and price Click "Create Product" Expected: "Product created successfully" message Token is automatically attached to request Test Logout Click "Logout" button in navigation Expected: Token cleared from localStorage, redirected to login Conclusion This project successfully demonstrates enterprise-level integration between React and Spring Boot with:

Proper separation of concerns Industry-standard authentication (JWT) Global CORS configuration Comprehensive error handling User-friendly feedback Security best practices The implementation covers both synchronous (Fetch) and async (Axios with interceptors) HTTP approaches, making it a complete learning resource for full-stack development. "# exp8_fsd"
