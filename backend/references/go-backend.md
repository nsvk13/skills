# Go Backend — chi, pgx, graceful shutdown

## Структура Go проекта

```
cmd/server/main.go       # wire-up зависимостей, запуск
internal/
  config/config.go       # env parsing
  service/               # бизнес-логика
  repository/
    postgres/            # pgx реализация
    memory/              # для тестов
  api/
    handler/             # тонкие хендлеры
    middleware/
migrations/
```

## main.go — graceful shutdown

```go
func main() {
    cfg, err := config.Load()
    if err != nil { log.Fatal(err) }

    logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))

    pool, err := pgxpool.New(ctx, cfg.DatabaseURL)
    if err != nil { log.Fatal(err) }
    defer pool.Close()

    store := postgres.NewUserStore(pool)
    svc := service.NewUserService(store, logger)
    router := api.NewRouter(svc, logger)

    srv := &http.Server{
        Addr:         fmt.Sprintf(":%d", cfg.Port),
        Handler:      router,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  120 * time.Second,
    }

    ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
    defer stop()

    go func() {
        logger.Info("starting", "addr", srv.Addr)
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            logger.Error("server error", "err", err)
            os.Exit(1)
        }
    }()

    <-ctx.Done()
    shutCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    srv.Shutdown(shutCtx)
    logger.Info("stopped")
}
```

## chi router

```go
func NewRouter(svc *service.UserService, log *slog.Logger) http.Handler {
    r := chi.NewRouter()

    r.Use(middleware.RequestID)
    r.Use(middleware.RealIP)
    r.Use(middleware.Logger)
    r.Use(middleware.Recoverer)
    r.Use(middleware.Timeout(30 * time.Second))

    r.Get("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })

    r.Route("/api/v1", func(r chi.Router) {
        r.Use(authMiddleware(log))
        r.Route("/users", func(r chi.Router) {
            r.Get("/{id}", handleGetUser(svc))
            r.Post("/", handleCreateUser(svc))
        })
    })

    return r
}

func handleGetUser(svc *service.UserService) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        id := chi.URLParam(r, "id")
        user, err := svc.GetUser(r.Context(), id)
        if err != nil {
            var notFound *NotFoundError
            if errors.As(err, &notFound) {
                writeJSON(w, http.StatusNotFound, map[string]string{"error": "not found"})
                return
            }
            writeJSON(w, http.StatusInternalServerError, map[string]string{"error": "internal"})
            return
        }
        writeJSON(w, http.StatusOK, user)
    }
}
```

## pgx — работа с PostgreSQL

```go
import "github.com/jackc/pgx/v5/pgxpool"

type UserStore struct { pool *pgxpool.Pool }

func (s *UserStore) FindByID(ctx context.Context, id string) (*User, error) {
    var u User
    err := s.pool.QueryRow(ctx,
        `SELECT id, name, email, created_at FROM users WHERE id = $1`, id,
    ).Scan(&u.ID, &u.Name, &u.Email, &u.CreatedAt)

    if errors.Is(err, pgx.ErrNoRows) {
        return nil, &NotFoundError{ID: id}
    }
    if err != nil {
        return nil, fmt.Errorf("findByID: %w", err)
    }
    return &u, nil
}

// Batch insert
func (s *UserStore) BulkCreate(ctx context.Context, users []User) error {
    batch := &pgx.Batch{}
    for _, u := range users {
        batch.Queue(`INSERT INTO users (id, name, email) VALUES ($1, $2, $3)`,
            u.ID, u.Name, u.Email)
    }
    return s.pool.SendBatch(ctx, batch).Close()
}

// Транзакция
func (s *UserStore) CreateWithSubscription(ctx context.Context, u User, plan string) error {
    return pgx.BeginFunc(ctx, s.pool, func(tx pgx.Tx) error {
        _, err := tx.Exec(ctx, `INSERT INTO users (id, name, email) VALUES ($1, $2, $3)`,
            u.ID, u.Name, u.Email)
        if err != nil {
            return fmt.Errorf("insert user: %w", err)
        }
        _, err = tx.Exec(ctx, `INSERT INTO subscriptions (user_id, plan) VALUES ($1, $2)`,
            u.ID, plan)
        return err
    })
}
```

## Config (caarlos0/env)

```go
type Config struct {
    Port        int    `env:"PORT" envDefault:"8080"`
    DatabaseURL string `env:"DATABASE_URL,required"`
    JWTSecret   string `env:"JWT_SECRET,required"`
    LogLevel    string `env:"LOG_LEVEL" envDefault:"info"`
}

func Load() (*Config, error) {
    var cfg Config
    if err := env.Parse(&cfg); err != nil {
        return nil, fmt.Errorf("parse config: %w", err)
    }
    return &cfg, nil
}
```