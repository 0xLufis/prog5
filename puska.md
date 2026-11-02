  1. Spring Boot MVC Fundamentals & Profiles (CRUD, In-Memory repositories, @Profile, @Qualifier, @PostConstruct)
  2. Jakarta Bean Validation & Error Handling (@Valid, BindingResult, custom validation rules)
  3. Spring Security 6+ & Authentication (SecurityFilterChain, dual filter chains for Web vs API, UserDetailsService,
  auto-login, method security)
  4. Thymeleaf & Internationalization (i18n) (th:replace vs th:insert, dynamic fragments *{__${name}__},
  LocaleResolver, LocaleChangeInterceptor)
  5. JSP / JSTL Legacy View Engine (JSTL tags, Spring form taglib, WAR packaging)
  6. Vaadin 24/25 Full-Stack Java UI (@Route, AppLayout/RouterLayout, Grid, ConfirmDialog, BeanValidationBinder,
  VaadinSecurityConfigurer)
  ──────
  ## Part 1: What to Expect in Your Upcoming Test

  ### Typical Exam Task Formats

  1. Implement or Complete a CRUD Feature:
      • Create or update an entity model with Jakarta validation annotations.
      • Write a @Service using an in-memory Map<Long, Entity> and AtomicLong.
      • Build a @Controller handling List, Create, Edit, and Delete actions with appropriate HTTP verbs (GET / POST),
      redirects, and validation checks.
  2. Configure Security & Authentication:
      • Set up SecurityFilterChain permitting public assets (/css/**, /login, /register) and locking down /cars/** or
      /admin/**.
      • Implement dual security chains (@Order(1) stateless HTTP Basic auth for /api/** with CSRF disabled, @Order(2)
      stateful form login for web pages).
      • Implement custom UserDetailsService (storing users in a ConcurrentHashMap with BCrypt hashing).
      • Auto-login a user programmatically upon registration via SecurityContextRepository.
      • Restrict access via method security annotations (@PreAuthorize, @RolesAllowed, @PostFilter).
  3. Template Engine (Thymeleaf / JSP) Tasks:
      • Implement forms bound to model attributes (th:object, th:field="*{...}", th:errors="*{...}").
      • Create reusable fragments (e.g., dynamic input field generator using Thymeleaf preprocessing *{__${name}__}).
      • Add language switching (EN / HU) using SessionLocaleResolver and LocaleChangeInterceptor.
  4. Vaadin View Implementation:
      • Build a view class extending VerticalLayout implementing BeforeEnterObserver and HasDynamicTitle.
      • Setup a Grid<T> with custom component action columns (Edit button navigating to route, Delete button opening
      a ConfirmDialog).
      • Setup a BeanValidationBinder<T> with readBean() and writeBeanIfValid().

  ──────
  ### High-Yield Traps & Trick Questions to Watch Out For

   Category            │ Common Trap / Gotcha                         │ Correct Approach
  ─────────────────────┼──────────────────────────────────────────────┼──────────────────────────────────────────────
   Validation          │ Placing Model or other parameters between    │ BindingResult must immediately follow the
                       │ @Valid ModelAttribute and BindingResult      │ @Valid model attribute in the method
                       │                                              │ parameters. Otherwise, Spring throws a 400
                       │                                              │ or 500 error instead of populating
                       │                                              │ validation errors.
   Spring Security     │ roles("ADMIN") vs hasAuthority("ADMIN")      │ roles("ADMIN") appends ROLE_ under the hood
                       │                                              │ (ROLE_ADMIN). hasRole("ADMIN") checks for
                       │                                              │ ROLE_ADMIN. If you specify
                       │                                              │ .hasAuthority("ADMIN") or
                       │                                              │ .authorities("ADMIN"), the ROLE_ prefix is
                       │                                              │ not added.
   Security 6 Session  │ SecurityContextHolder.setContext(context)    │ In Spring Security 6 / Spring Boot 3+, you
                       │ doesn't persist across redirects             │ must explicitly save the context to the
                       │                                              │ session using
                       │                                              │ securityContextRepository.saveContext(contex
                       │                                              │ t, request, response).
   JSP in Boot         │ JSP returns blank or 404                     │ JSP requires <packaging>war</packaging>,
                       │                                              │ tomcat-embed-jasper (provided), and
                       │                                              │ spring.mvc.view.prefix / suffix configured
                       │                                              │ in application.yaml.
   Thymeleaf Fragments │ th:replace vs th:insert                      │ th:replace replaces the host tag with the
                       │                                              │ fragment tag. th:insert places the fragment
                       │                                              │ inside the host tag.
   Vaadin Grid         │ new Grid<>(Car.class) creates all columns    │ Pass false: new Grid<>(Car.class, false) to
                       │ automatically                                │ manually control and order columns.
   Vaadin Routing      │ Route parameters not accessible in           │ In Vaadin, URL path parameters like
                       │ constructor                                  │ @Route("cars/:id/edit") must be extracted in
                       │                                              │ beforeEnter(BeforeEnterEvent event) using
                       │                                              │ event.getRouteParameters().
  ──────
  ## Part 2: Complete Annotated Cheat Sheet

  ### 1. Model with Jakarta Validation & Lombok

  File reference: Car.java

    package hu.mik.pte.prog5.crud_mvc.model;

    import jakarta.validation.constraints.Min;
    import jakarta.validation.constraints.NotBlank;
    import jakarta.validation.constraints.NotNull;
    import jakarta.validation.constraints.Size;
    import lombok.*;

    @Getter
    @Setter
    @ToString
    @NoArgsConstructor       // Required for Spring MVC form binding and JSON deserialization
    @AllArgsConstructor
    public class Car {
        private Long id;

        // Checks that string is not null and trimmed length is > 0
        @NotBlank(message = "{validation.notBlank}") // Can reference translation keys
        @Size(min = 2, max = 50, message = "Make must be between 2 and 50 characters")
        private String make;

        @NotBlank(message = "Model cannot be blank")
        private String model;

        @NotNull(message = "Year is mandatory")
        @Min(value = 1950, message = "Year must not be before 1950")
        private Integer manufactureYear;

        private String owner; // Used for @PostFilter demonstration
    }
  ──────
  ### 2. Service Layer: In-Memory Storage, Profiles & Method Security

  File references:

  • CarService.java
  • InMemoryDataService.java
  • InMemoryMockDataService.java

  #### Profile-dependent Data Sources

    public interface DataService {
        Collection<Car> getValues();
    }

    // Activated when spring.profiles.active=plain
    @Service("plain")
    @Profile("plain")
    public class InMemoryDataService implements DataService {
        @Override
        public Collection<Car> getValues() {
            return List.of(
                new Car(1L, "Skoda", "Octavia", 2021, "user"),
                new Car(2L, "Toyota", "Corolla", 2022, "admin")
            );
        }
    }

    // Activated when spring.profiles.active=mock
    @Service("mock")
    @Profile("mock")
    public class InMemoryMockDataService implements DataService {
        @Override
        public Collection<Car> getValues() {
            return List.of(
                new Car(1L, "Ford", "Focus", 2020, "user"),
                new Car(2L, "Ford", "Fiesta", 2019, "user")
            );
        }
    }

  #### CarService with Method Security

    @Service
    @RequiredArgsConstructor
    public class CarService {

        private final Map<Long, Car> carList = new ConcurrentHashMap<>();
        private final AtomicLong idSeq = new AtomicLong(0);
        private final DataService dataService; // Injected based on active profile

        @PostConstruct // Runs automatically after bean initialization
        private void init() {
            this.dataService.getValues().forEach(this::save);
        }

        public Car save(Car car) {
            if (car.getId() == null) {
                car.setId(this.idSeq.incrementAndGet()); // Thread-safe auto-increment
            }
            this.carList.put(car.getId(), car);
            return car;
        }

        // METHOD SECURITY DEMONSTRATED IN LAB:
        // @PreAuthorize("hasRole('ADMIN')")              // Must be ADMIN before invocation
        // @Secured("ROLE_ADMIN")                         // Spring legacy annotation (requires ROLE_)
        // @RolesAllowed("ADMIN")                         // Standard JSR-250 annotation
        // @PostFilter("filterObject.owner == authentication.name or hasRole('ADMIN')") // Filters returned
  collection
        public Collection<Car> list() {
            // Programmatic check of current user:
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            String currentUsername = auth != null ? auth.getName() : null;

            return new ArrayList<>(this.carList.values());
        }

        public Car findById(Long id) {
            return this.carList.get(id);
        }

        // @PreAuthorize("hasRole('ADMIN')")
        public void deleteById(Long id) {
            this.carList.remove(id);
        }
    }
  ──────
  ### 3. Spring MVC Controller (CRUD Lifecycle)

  File reference: CarController.java

    @Controller
    @RequestMapping("/cars")
    @RequiredArgsConstructor
    public class CarController {

        private final CarService carService;

        // 1. LIST: GET /cars
        @GetMapping
        public String list(Model model) {
            model.addAttribute("cars", this.carService.list());
            return "cars/list"; // Resolves to templates/cars/list.html or WEB-INF/views/cars/list.jsp
        }

        // 2. NEW FORM: GET /cars/new
        @GetMapping("/new")
        public String newForm(Model model) {
            model.addAttribute("car", new Car()); // Blank object to bind form fields
            return "cars/form";
        }

        // 3. CREATE: POST /cars
        @PostMapping
        public String create(
                @Valid @ModelAttribute("car") Car car,
                BindingResult bindingResult // MUST be adjacent to @Valid Car
        ) {
            if (bindingResult.hasErrors()) {
                return "cars/form"; // Return to form view showing errors, DO NOT redirect
            }
            this.carService.save(car);
            return "redirect:/cars"; // Post-Redirect-Get (PRG) pattern
        }

        // 4. EDIT FORM: GET /cars/{id}/edit
        @GetMapping("/{id}/edit")
        public String editForm(@PathVariable Long id, Model model) {
            Car car = this.carService.findById(id);
            model.addAttribute("car", car);
            return "cars/form";
        }

        // 5. UPDATE: POST /cars/{id}
        @PostMapping("/{id}")
        public String update(
                @PathVariable Long id,
                @Valid @ModelAttribute("car") Car car,
                BindingResult bindingResult
        ) {
            if (bindingResult.hasErrors()) {
                return "cars/form";
            }
            car.setId(id);
            this.carService.save(car);
            return "redirect:/cars";
        }

        // 6. DELETE: POST /cars/{id}/delete
        @PostMapping("/{id}/delete")
        public String delete(@PathVariable Long id) {
            this.carService.deleteById(id);
            return "redirect:/cars";
        }
    }
  ──────
  ### 4. Spring Security 6: Dual Filter Chains, UserDetails & Auto-Login

  File reference: SecurityConfig.java

    @Configuration
    @EnableWebSecurity(debug = false)
    @EnableMethodSecurity(securedEnabled = true, jsr250Enabled = true)
    public class SecurityConfig {

        // CHAIN 1: Stateless REST API with HTTP Basic Auth
        @Bean
        @Order(1) // High priority
        public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
            http.securityMatcher("/api/**") // Only matches /api/** paths
                .authorizeHttpRequests(auth -> auth.anyRequest().authenticated())
                .httpBasic(Customizer.withDefaults())
                .formLogin(AbstractHttpConfigurer::disable)
                .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .csrf(AbstractHttpConfigurer::disable);

            return http.build();
        }

        // CHAIN 2: Stateful Web MVC Application with Form Login
        @Bean
        @Order(2) // Fallback for all other web requests
        public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
            http.authorizeHttpRequests(auth -> auth
                    .requestMatchers("/css/**", "/js/**", "/images/**", "/webjars/**").permitAll()
                    .requestMatchers("/error").permitAll()
                    // Essential for forward/error dispatches in JSP/MVC:
                    .dispatcherTypeMatchers(DispatcherType.FORWARD, DispatcherType.ERROR).permitAll()
                    .requestMatchers("/register", "/login").permitAll()
                    // .requestMatchers(HttpMethod.POST, "/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
                )
                .formLogin(login -> login
                    .loginPage("/login")               // Custom login GET route
                    .defaultSuccessUrl("/cars", true)  // Where to send user on success
                    .failureUrl("/login?error")        // URL on invalid credentials
                )
                .logout(logout -> logout
                    .logoutUrl("/logout")
                    .logoutSuccessUrl("/login?logout") // Displays logout message
                )
                .httpBasic(AbstractHttpConfigurer::disable);

            return http.build();
        }

        @Bean
        public PasswordEncoder passwordEncoder() {
            return new BCryptPasswordEncoder();
        }
    }

  #### Custom UserDetailsService with In-Memory Concurrent Storage

  File reference: AppUserDetailsService.java

    public record AppUser(String username, String passwordHash, Set<String> roles) {}

    @Service
    public class AppUserDetailsService implements UserDetailsService {

        private final Map<String, AppUser> appUsers = new ConcurrentHashMap<>();
        private final PasswordEncoder passwordEncoder;

        public AppUserDetailsService(PasswordEncoder encoder) {
            this.passwordEncoder = encoder;
            // Seed default users:
            this.appUsers.put("user", new AppUser("user", encoder.encode("user"), Set.of("USER")));
            this.appUsers.put("admin", new AppUser("admin", encoder.encode("admin"), Set.of("USER", "ADMIN")));
        }

        public boolean exists(String username) {
            return this.appUsers.containsKey(username);
        }

        public void register(String username, String rawPassword) {
            AppUser user = new AppUser(username, this.passwordEncoder.encode(rawPassword), Set.of("USER"));
            if (this.appUsers.putIfAbsent(username, user) != null) {
                throw new IllegalStateException("User already exists!");
            }
        }

        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            AppUser user = this.appUsers.get(username);
            if (user == null) {
                throw new UsernameNotFoundException("Not found: " + username);
            }
            return User.builder()
                    .username(user.username())
                    .password(user.passwordHash())
                    .roles(user.roles().toArray(String[]::new)) // Automatically adds ROLE_
                    .build();
        }
    }

  #### Registration Controller with Manual Validation & Spring Security 6 Auto-Login

  File reference: RegisterController.java

    @Controller
    @RequiredArgsConstructor
    public class RegisterController {

        private final AppUserDetailsService userDetailsService;
        // Repository needed to persist authenticated session across redirects:
        private final SecurityContextRepository securityContextRepo = new HttpSessionSecurityContextRepository();

        @GetMapping("/register")
        public String registerForm(Model model) {
            model.addAttribute("registerForm", new RegisterForm());
            return "register";
        }

        @PostMapping("/register")
        public String register(
                @Valid @ModelAttribute("registerForm") RegisterForm form,
                BindingResult bindingResult,
                HttpServletRequest request,
                HttpServletResponse response
        ) {
            // Custom validations:
            if (!form.getPassword().equals(form.getConfirmPassword())) {
                bindingResult.rejectValue("confirmPassword", "mismatch", "Passwords do not match");
            }
            if (this.userDetailsService.exists(form.getUsername())) {
                bindingResult.rejectValue("username", "taken", "Username already exists");
            }
            if (bindingResult.hasErrors()) {
                return "register";
            }

            // 1. Save new user
            this.userDetailsService.register(form.getUsername(), form.getPassword());

            // 2. Programmatic authentication (Auto-login)
            UserDetails userDetails = this.userDetailsService.loadUserByUsername(form.getUsername());
            Authentication auth = UsernamePasswordAuthenticationToken.authenticated(
                    userDetails, null, userDetails.getAuthorities()
            );

            // 3. Store into security context and persist in session
            SecurityContext context = SecurityContextHolder.createEmptyContext();
            context.setAuthentication(auth);
            SecurityContextHolder.setContext(context);
            this.securityContextRepo.saveContext(context, request, response);

            return "redirect:/cars";
        }
    }
  ──────
  ### 5. Internationalization (i18n) Configuration

  File reference: WebConfig.java

    @Configuration
    public class WebConfig implements WebMvcConfigurer {

        // Resolves and stores current locale in user HTTP session
        @Bean
        public LocaleResolver localeResolver() {
            SessionLocaleResolver resolver = new SessionLocaleResolver();
            resolver.setDefaultLocale(Locale.of("en")); // Default language
            return resolver;
        }

        // Intercepts URL requests with ?lang=hu or ?lang=en and updates session
        @Override
        public void addInterceptors(InterceptorRegistry registry) {
            LocaleChangeInterceptor interceptor = new LocaleChangeInterceptor();
            interceptor.setParamName("lang");
            registry.addInterceptor(interceptor);
        }
    }

  In application.yaml:

    spring:
      messages:
        fallback-to-system-locale: false
        basename: messages

  In messages_hu.properties:

    car.list.title=Autó lista
    locale.hu=Magyar
    locale.en=Angol

  In messages_en.properties:

    car.list.title=Car list
    locale.hu=Hungarian
    locale.en=English
  ──────
  ### 6. Thymeleaf Templates & Advanced Fragments

  File references:

  • layout.html
  • form-fields.html
  • list.html
  • form.html

  #### Base Layout Fragment (fragments/layout.html)

    <!DOCTYPE html>
    <html lang="en" xmlns:th="https://www.thymeleaf.org"
                    xmlns:sec="https://www.thymeleaf.org/extras/spring-security">

    <head th:fragment="head(title)">
        <meta charset="UTF-8">
        <title th:text="${title}">App</title>
        <link rel="stylesheet" th:href="@{/css/style.css}">
    </head>
    <body>
    <header th:fragment="header(pageName)">
        <!-- Language Switcher Links -->
        <a th:href="@{''(lang='en')}">EN</a>
        <a th:href="@{''(lang='hu')}">HU</a>

        <!-- Language Switcher Dropdown -->
        <form method="get">
            <select name="lang" onchange="this.form.submit()">
                <option value="hu" th:text="#{locale.hu}" th:selected="${#locale.language == 'hu'}"></option>
                <option value="en" th:text="#{locale.en}" th:selected="${#locale.language == 'en'}"></option>
            </select>
        </form>

        <!-- Spring Security integration -->
        <th:block sec:authorize="isAuthenticated()">
            <span>Logged in as: <b sec:authentication="name">User</b></span>
            <form th:action="@{/logout}" method="post" style="display:inline;">
                <button type="submit">Logout</button>
            </form>
        </th:block>
    </header>
    </body>
    </html>

  #### Reusable Input Field Fragment with Preprocessor Expression (fragments/form-fields.html)

    <!DOCTYPE html>
    <html lang="en" xmlns:th="https://www.thymeleaf.org">
    <body>
      <!-- Note the __${name}__ preprocessor syntax that dynamically evaluates field name inside *{...} -->
      <div th:fragment="field(name, label, type)">
          <label th:for="${name}" th:text="${label}"></label>
          <input th:type="${type}" th:id="${name}" th:field="*{__${name}__}"/>
          <span class="error" th:errors="*{__${name}__}"></span>
      </div>
    </body>
    </html>

  #### Car Form View (cars/form.html)

    <!DOCTYPE html>
    <html xmlns:th="https://www.thymeleaf.org">
    <head th:replace="~{fragments/layout :: head('Car Page')}"></head>
    <body>
    <header th:replace="~{fragments/layout :: header('Car Page')}"></header>

    <h1 th:text="${car.id == null ? 'New Car' : 'Edit Car'}"></h1>

    <!-- Dynamic action: POST to /cars when creating, /cars/{id} when updating -->
    <form class="card" method="post" th:object="${car}"
          th:action="${car.id == null} ? @{/cars} : @{/cars/{id}(id=${car.id})}">

        <!-- Regular field -->
        <label for="make">Make</label>
        <input type="text" id="make" th:field="*{make}"/>
        <span class="error" th:errors="*{make}"></span>

        <!-- Reusable fragment fields -->
        <div th:replace="~{fragments/form-fields :: field('model', 'Model', 'text')}"></div>
        <div th:replace="~{fragments/form-fields :: field('manufactureYear', 'Manufacture Year', 'number')}"></div>

        <p>
            <button type="submit">Save</button>
            <a class="button" th:href="@{/cars}">Cancel</a>
        </p>
    </form>
    </body>
    </html>

  #### Car Table List View (cars/list.html)

    <!DOCTYPE html>
    <html xmlns:th="https://www.thymeleaf.org">
    <head th:replace="~{fragments/layout :: head('Car List')}"></head>
    <body>
    <header th:replace="~{fragments/layout :: header('Car List')}"></header>

    <div class="top-bar">
        <h1 th:text="#{car.list.title}">Cars</h1>
        <a class="button" th:href="@{/cars/new}">+ New car</a>
    </div>

    <table>
        <thead>
            <tr>
                <th>Id</th><th>Make</th><th>Model</th><th>Year</th><th>Actions</th>
            </tr>
        </thead>
        <tbody>
            <tr th:if="${cars.isEmpty()}">
                <td colspan="5">No cars found.</td>
            </tr>
            <tr th:each="car : ${cars}">
                <td th:text="${car.id}">1</td>
                <td th:text="${car.make}">Skoda</td>
                <td th:text="${car.model}">Octavia</td>
                <td th:text="${car.manufactureYear}">2021</td>
                <td class="actions">
                    <a class="button" th:href="@{/cars/{id}/edit(id=${car.id})}">Edit</a>
                    <form th:action="@{/cars/{id}/delete(id=${car.id})}" method="post"
                          onsubmit="return confirm('Are you sure?');" style="display:inline;">
                        <button type="submit" class="danger">Delete</button>
                    </form>
                </td>
            </tr>
        </tbody>
    </table>
    </body>
    </html>

  #### Custom Login Template with Error / Logout Messaging (login.html)

    <!DOCTYPE html>
    <html lang="en" xmlns:th="https://www.thymeleaf.org">
    <head th:replace="~{fragments/layout :: head('Login')}"></head>
    <body>
    <h1 th:text="'Login'"></h1>

    <!-- param.error and param.logout check URL query parameters -->
    <p class="error" th:if="${param.error}" style="color:red;">Invalid username or password.</p>
    <p class="success" th:if="${param.logout}" style="color:green;">Logged out successfully.</p>

    <form th:action="@{/login}" method="post" class="card">
        <label for="username">Username</label>
        <input type="text" id="username" name="username" required>

        <label for="password">Password</label>
        <input type="password" id="password" name="password" required>

        <button type="submit">Login</button>
    </form>
    <p>No account yet? <a th:href="@{/register}">Register here</a></p>
    </body>
    </html>
  ──────
  ### 7. Vaadin Framework: Modern Pure-Java Web UI

  File references:

  • SecurityConfig.java (Vaadin)
  • MainLayout.java
  • CarListView.java
  • CarFormView.java
  • LoginView.java

  #### Vaadin Security Configuration

    @Configuration
    @EnableWebSecurity
    @EnableMethodSecurity
    public class SecurityConfig {

        @Bean
        public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
            // Delegates security configuration to Vaadin's internal helper:
            http.with(VaadinSecurityConfigurer.vaadin(), cfg -> cfg.loginView(LoginView.class));
            return http.build();
        }

        @Bean
        public PasswordEncoder passwordEncoder() {
            return new BCryptPasswordEncoder();
        }

        @Bean
        public UserDetailsService users(PasswordEncoder enc) {
            return new InMemoryUserDetailsManager(
                User.withUsername("user").password(enc.encode("user")).roles("USER").build(),
                User.withUsername("admin").password(enc.encode("admin")).roles("ADMIN", "USER").build()
            );
        }
    }

  #### Vaadin RouterLayout with AuthenticationContext (MainLayout.java)

    @AnonymousAllowed
    public class MainLayout extends VerticalLayout implements RouterLayout {

        private final Div content = new Div();
        private final AuthenticationContext authContext; // Injected by Vaadin Flow Spring Security

        public MainLayout(AuthenticationContext authContext) {
            this.authContext = authContext;

            setSizeFull();
            setMaxWidth("900px");
            getStyle().set("margin", "0 auto");

            content.setWidthFull();
            add(createHeader(), content, createFooter());
            setFlexGrow(1, content);
        }

        private Header createHeader() {
            H3 title = new H3(new RouterLink(getTranslation("title"), CarListView.class));

            // Language Switcher:
            ComboBox<Locale> langSelect = new ComboBox<>();
            langSelect.setItems(Locale.of("hu"), Locale.of("en"));
            langSelect.setItemLabelGenerator(l -> l.getDisplayLanguage(getLocale()));
            langSelect.setValue(UI.getCurrent().getLocale());
            langSelect.addValueChangeListener(e -> {
                VaadinSession.getCurrent().setLocale(e.getValue());
                UI.getCurrent().getPage().reload();
            });

            HorizontalLayout bar = new HorizontalLayout(title, langSelect);
            bar.setWidthFull();
            bar.expand(title);

            if (this.authContext.isAuthenticated()) {
                String name = this.authContext.getPrincipalName().orElse("");
                Button logout = new Button(getTranslation("header.logout"), e -> this.authContext.logout());
                bar.add(new Span(name), logout);
            }

            return new Header(bar);
        }

        private Footer createFooter() {
            return new Footer(new Span(getTranslation("app.footer")));
        }

        // Called when child view route changes:
        @Override
        public void showRouterLayoutContent(HasElement childContent) {
            this.content.removeAll();
            if (childContent != null) {
                this.content.getElement().appendChild(childContent.getElement());
            }
        }
    }

  #### Vaadin Login View (LoginView.java)

    @Route(value = "login", layout = MainLayout.class)
    @AnonymousAllowed
    public class LoginView extends VerticalLayout implements HasDynamicTitle {

        private final LoginForm loginForm = new LoginForm();

        public LoginView() {
            this.loginForm.setAction("login"); // Submits to Spring Security's filter
            this.loginForm.setForgotPasswordButtonVisible(false);
            add(this.loginForm);
            setAlignItems(Alignment.CENTER);
        }

        @Override
        public String getPageTitle() {
            return getTranslation("login.title");
        }
    }

  #### Vaadin List View with Grid & ConfirmDialog (CarListView.java)

    @Route(value = "cars", layout = MainLayout.class)
    @RouteAlias(value = "", layout = MainLayout.class) // Default landing route
    @PermitAll
    public class CarListView extends VerticalLayout implements HasDynamicTitle {

        private final CarService carService;
        // Set autoCreateColumns=false to define custom headers and column order
        private final Grid<Car> grid = new Grid<>(Car.class, false);

        public CarListView(CarService carService) {
            this.carService = carService;

            Button newBtn = new Button(getTranslation("car.new"),
                e -> UI.getCurrent().navigate(CarFormView.class));

            // Adding bound columns:
            this.grid.addColumn(Car::getMake).setHeader(getTranslation("car.make"));
            this.grid.addColumn(Car::getModel).setHeader(getTranslation("car.model"));
            this.grid.addColumn(Car::getManufactureYear).setHeader(getTranslation("car.manufactureYear"));

            // Adding component actions column:
            this.grid.addComponentColumn(this::createActions);
            this.grid.setEmptyStateText(getTranslation("car.list.empty"));

            add(newBtn, this.grid);
            refresh();
        }

        private HorizontalLayout createActions(Car car) {
            Button editBtn = new Button(getTranslation("car.edit"),
                e -> UI.getCurrent().navigate("cars/" + car.getId() + "/edit"));

            Button deleteBtn = new Button(getTranslation("car.delete"), e -> {
                ConfirmDialog dialog = new ConfirmDialog();
                dialog.setHeader(getTranslation("car.delete"));
                dialog.setText("Are you sure you want to delete this car?");
                dialog.setCancelable(true);
                dialog.setCancelText(getTranslation("car.cancel"));
                dialog.addConfirmListener(ev -> {
                    this.carService.deleteById(car.getId());
                    refresh();
                });
                dialog.open();
            });

            return new HorizontalLayout(editBtn, deleteBtn);
        }

        private void refresh() {
            this.grid.setItems(this.carService.list());
        }

        @Override
        public String getPageTitle() {
            return getTranslation("car.list.title");
        }
    }

  #### Vaadin Form View with BeanValidationBinder (CarFormView.java)

    @Route(value = "cars/new", layout = MainLayout.class)
    @RouteAlias(value = "cars/:id/edit", layout = MainLayout.class) // :id route parameter
    @PermitAll
    public class CarFormView extends VerticalLayout implements BeforeEnterObserver, HasDynamicTitle {

        private final CarService carService;
        private Car car;

        // Automatically honors @NotBlank, @Min, @NotNull on Car entity:
        private final BeanValidationBinder<Car> binder = new BeanValidationBinder<>(Car.class);

        private final H1 title = new H1();
        private final TextField make = new TextField(getTranslation("car.make"));
        private final TextField model = new TextField(getTranslation("car.model"));
        private final IntegerField manufactureYear = new IntegerField(getTranslation("car.manufactureYear"));

        public CarFormView(CarService carService) {
            this.carService = carService;

            // Bind UI input fields to bean properties:
            this.binder.forField(this.make).bind("make");
            this.binder.forField(this.model).bind("model");
            this.binder.forField(this.manufactureYear).bind("manufactureYear");

            FormLayout formLayout = new FormLayout(this.make, this.model, this.manufactureYear);

            Button saveBtn = new Button(getTranslation("car.save"), e -> save());
            Button cancelBtn = new Button(getTranslation("car.cancel"),
                e -> UI.getCurrent().navigate(CarListView.class));

            add(this.title, formLayout, new HorizontalLayout(saveBtn, cancelBtn));
        }

        // Lifecycle hook: Extracts route parameter BEFORE view enters DOM
        @Override
        public void beforeEnter(BeforeEnterEvent event) {
            this.car = event.getRouteParameters()
                    .getLong("id")
                    .map(this.carService::findById)
                    .orElseGet(Car::new);

            this.binder.readBean(this.car); // Populate form inputs with bean values
            this.title.setText(getPageTitle());
        }

        private void save() {
            // Validates input against constraints; writes to bean if valid:
            if (this.binder.writeBeanIfValid(this.car)) {
                this.carService.save(this.car);
                UI.getCurrent().navigate(CarListView.class);
            }
        }

        @Override
        public String getPageTitle() {
            return getTranslation(this.car == null || this.car.getId() == null
                ? "car.new.title"
                : "car.edit.title");
        }
    }
  ──────
  ### 8. JSP & JSTL (Legacy View Engine) Reference

  File references:

  • list.jsp
  • form.jsp

  #### JSTL Table Rendering (list.jsp)

    <%@ page contentType="text/html;charset=UTF-8" language="java" %>
    <%@ taglib prefix="c" uri="jakarta.tags.core" %> <%-- Jakarta EE taglib URI --%>
    <!DOCTYPE html>
    <html>
    <head>
        <title>Cars</title>
        <link rel="stylesheet" href="${pageContext.request.contextPath}/css/style.css">
    </head>
    <body>
        <h1>Cars</h1>
        <a href="${pageContext.request.contextPath}/cars/new">+ New Car</a>

        <table>
            <tbody>
            <c:forEach var="car" items="${cars}">
                <tr>
                    <td>${car.id}</td>
                    <td>${car.make}</td>
                    <td>${car.model}</td>
                    <td>${car.manufactureYear}</td>
                    <td>
                        <a href="${pageContext.request.contextPath}/cars/${car.id}/edit">Edit</a>
                        <form action="${pageContext.request.contextPath}/cars/${car.id}/delete" method="post"
                              onsubmit="return confirm('Delete this car?');">
                            <button type="submit">Delete</button>
                        </form>
                    </td>
                </tr>
            </c:forEach>
            <c:if test="${empty cars}">
                <tr><td colspan="5">No cars yet.</td></tr>
            </c:if>
            </tbody>
        </table>
    </body>
    </html>

  #### Spring Form Taglib (form.jsp)

    <%@ page contentType="text/html;charset=UTF-8" language="java" %>
    <%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
    <!DOCTYPE html>
    <html>
    <head>
        <title>${car.id == null ? 'New car' : 'Edit car'}</title>
    </head>
    <body>
        <form:form class="card" modelAttribute="car" method="post"
                   action="${pageContext.request.contextPath}/cars${car.id == null ? '' : '/'.concat(car.id)}">

            <label for="make">Make</label>
            <form:input path="make" id="make"/>
            <form:errors path="make" cssClass="error"/>

            <label for="model">Model</label>
            <form:input path="model" id="model"/>
            <form:errors path="model" cssClass="error"/>

            <label for="manufactureYear">Manufacture Year</label>
            <form:input path="manufactureYear" id="manufactureYear" type="number"/>
            <form:errors path="manufactureYear" cssClass="error"/>

            <input type="submit" value="Save"/>
            <a href="${pageContext.request.contextPath}/cars">Cancel</a>
        </form:form>
    </body>
    </html>
  ──────
  ## Part 3: Quick Recall Sheet (Summary of Key Annotations & Syntaxes)

   Purpose          │ Spring MVC / Security         │ Thymeleaf                     │ Vaadin Flow
  ──────────────────┼───────────────────────────────┼───────────────────────────────┼────────────────────────────────
   Route / Path     │ @GetMapping("/cars"),         │ th:href="@{/cars}",           │ @Route("cars"),
                    │ @PostMapping                  │ th:action="@{/cars}"          │ @RouteAlias("")
   Path Variable    │ @PathVariable Long id         │ @{/cars/{id}(id=${c.id})}     │ @Route("cars/:id/edit"),
                    │                               │                               │ event.getRouteParameters()
   Model Binding    │ @ModelAttribute("car") Car    │ th:object="${car}",           │ BeanValidationBinder<Car>,
                    │ car                           │ th:field="*{make}"            │ binder.bind(...)
   Show Errors      │ bindingResult.hasErrors()     │ th:errors="*{make}"           │ Automatically handled by
                    │                               │                               │ BeanValidationBinder
   Iteration        │ In Controller:                │ <tr th:each="c : ${cars}">    │ grid.setItems(collection)
                    │ model.addAttribute(...)       │                               │
   Conditional      │ In Java: if (...)             │ th:if="${c.id == null}"       │ Component .setVisible(boolean)
   Translations     │ messageSource.getMessage(...) │ #{key}                        │ getTranslation("key")
   Auth Tag / Check │ @PreAuthorize("hasRole('ADMIN │ <th:block                     │ @RolesAllowed("ADMIN"),
                    │ ')")                          │ sec:authorize="hasRole('ADMIN │ @PermitAll, @AnonymousAllowed
                    │                               │ ')">                          │
