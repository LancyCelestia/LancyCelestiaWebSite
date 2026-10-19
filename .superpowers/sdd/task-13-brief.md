### Task 13: 自研投稿服务 `submitd`（Go + SQLite）

**Files:**
- Create: `server/submitd/{go.mod,main.go,store.go,handlers.go,store_test.go,handlers_test.go}`
- Test: `cd server/submitd && go test ./...`

**Interfaces:**
- Produces: `POST /posts` 创建投稿；`GET /posts` 列出（需管理员）；`GET /healthz`
- 端口 `23367`

> **架构决策**：Artalk **没有**投稿能力（B 审查确认其 README 与 HTTP API 均无 posts/投稿实体）。用户选 **2B 自研**。本服务独立于 Artalk，内存约 20–30 MB。

- [ ] **Step 1: 写失败测试 `store_test.go`**

```go
package main

import "testing"

func TestCreateAndList(t *testing.T) {
	s, err := OpenStore(":memory:")
	if err != nil { t.Fatal(err) }
	defer s.Close()

	id, err := s.CreatePost("标题", "正文内容", "澜汐")
	if err != nil { t.Fatal(err) }
	if id == 0 { t.Fatal("expected non-zero id") }

	posts, err := s.ListPosts()
	if err != nil { t.Fatal(err) }
	if len(posts) != 1 { t.Fatalf("want 1 post, got %d", len(posts)) }
	if posts[0].Title != "标题" { t.Fatalf("title mismatch: %s", posts[0].Title) }
}

func TestRejectEmpty(t *testing.T) {
	s, _ := OpenStore(":memory:")
	defer s.Close()
	if _, err := s.CreatePost("", "x", "a"); err == nil {
		t.Fatal("expected error for empty title")
	}
}
```

- [ ] **Step 2: 运行确认失败**

Run: `cd server/submitd && go mod init submitd && go test ./...`
Expected: 编译失败（`OpenStore` 未定义）

- [ ] **Step 3: 写 `store.go`**

```go
package main

import (
	"database/sql"
	"errors"
	"strings"
	"time"

	_ "modernc.org/sqlite"
)

type Post struct {
	ID        int64
	Title     string
	Body      string
	Author    string
	CreatedAt time.Time
}

type Store struct{ db *sql.DB }

func OpenStore(dsn string) (*Store, error) {
	db, err := sql.Open("sqlite", dsn)
	if err != nil { return nil, err }
	_, err = db.Exec(`CREATE TABLE IF NOT EXISTS posts (
		id INTEGER PRIMARY KEY AUTOINCREMENT,
		title TEXT NOT NULL,
		body TEXT NOT NULL,
		author TEXT NOT NULL,
		created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
	)`)
	if err != nil { db.Close(); return nil, err }
	return &Store{db: db}, nil
}

func (s *Store) Close() error { return s.db.Close() }

func (s *Store) CreatePost(title, body, author string) (int64, error) {
	if strings.TrimSpace(title) == "" { return 0, errors.New("title required") }
	if strings.TrimSpace(body) == "" { return 0, errors.New("body required") }
	res, err := s.db.Exec(`INSERT INTO posts (title, body, author) VALUES (?, ?, ?)`, title, body, author)
	if err != nil { return 0, err }
	return res.LastInsertId()
}

func (s *Store) ListPosts() ([]Post, error) {
	rows, err := s.db.Query(`SELECT id, title, body, author, created_at FROM posts ORDER BY created_at DESC`)
	if err != nil { return nil, err }
	defer rows.Close()
	var out []Post
	for rows.Next() {
		var p Post
		if err := rows.Scan(&p.ID, &p.Title, &p.Body, &p.Author, &p.CreatedAt); err != nil {
			return nil, err
		}
		out = append(out, p)
	}
	return out, rows.Err()
}
```

- [ ] **Step 4: 运行测试确认通过**

Run: `cd server/submitd && go get modernc.org/sqlite && go test ./...`
Expected: `ok submitd`

- [ ] **Step 5: 写失败测试 `handlers_test.go`**

```go
package main

import (
	"net/http/httptest"
	"strings"
	"testing"
)

func TestHealthz(t *testing.T) {
	mux := newMux(mustStore(t))
	req := httptest.NewRequest("GET", "/healthz", nil)
	w := httptest.NewRecorder()
	mux.ServeHTTP(w, req)
	if w.Code != 200 { t.Fatalf("want 200, got %d", w.Code) }
}

func TestPostRequiresAuth(t *testing.T) {
	mux := newMux(mustStore(t))
	req := httptest.NewRequest("POST", "/posts", strings.NewReader(`{"title":"t","body":"b"}`))
	w := httptest.NewRecorder()
	mux.ServeHTTP(w, req)
	if w.Code != 401 { t.Fatalf("want 401 without token, got %d", w.Code) }
}

func mustStore(t *testing.T) *Store {
	s, err := OpenStore(":memory:")
	if err != nil { t.Fatal(err) }
	t.Cleanup(func() { s.Close() })
	return s
}
```

- [ ] **Step 6: 运行确认失败**

Run: `cd server/submitd && go test ./...`
Expected: `undefined: newMux`

- [ ] **Step 7: 写 `handlers.go` 与 `main.go`**

```go
// handlers.go
package main

import (
	"crypto/subtle"
	"encoding/json"
	"net/http"
	"os"
	"strings"
)

func newMux(s *Store) *http.ServeMux {
	mux := http.NewServeMux()

	mux.HandleFunc("GET /healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`{"ok":true}`))
	})

	mux.HandleFunc("POST /posts", func(w http.ResponseWriter, r *http.Request) {
		if !authed(r) { http.Error(w, "unauthorized", http.StatusUnauthorized); return }
		var in struct{ Title, Body, Author string }
		if err := json.NewDecoder(r.Body).Decode(&in); err != nil {
			http.Error(w, "bad json", http.StatusBadRequest); return
		}
		id, err := s.CreatePost(in.Title, in.Body, in.Author)
		if err != nil { http.Error(w, err.Error(), http.StatusBadRequest); return }
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]any{"id": id})
	})

	mux.HandleFunc("GET /posts", func(w http.ResponseWriter, r *http.Request) {
		if !authed(r) { http.Error(w, "unauthorized", http.StatusUnauthorized); return }
		posts, err := s.ListPosts()
		if err != nil { http.Error(w, "db error", http.StatusInternalServerError); return }
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(posts)
	})

	return mux
}

func authed(r *http.Request) bool {
	token := os.Getenv("SUBMITD_TOKEN")
	if token == "" { return false }
	got := strings.TrimPrefix(r.Header.Get("Authorization"), "Bearer ")
	return subtle.ConstantTimeCompare([]byte(got), []byte(token)) == 1
}
```

```go
// main.go
package main

import (
	"log"
	"net/http"
	"os"
	"time"
)

func main() {
	dsn := os.Getenv("SUBMITD_DSN")
	if dsn == "" { dsn = "/var/lib/submitd/submitd.db" }
	s, err := OpenStore(dsn)
	if err != nil { log.Fatal(err) }
	defer s.Close()

	srv := &http.Server{
		Addr:              ":23367",
		Handler:           newMux(s),
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       15 * time.Second,
		WriteTimeout:      15 * time.Second,
	}
	log.Println("submitd listening on :23367")
	log.Fatal(srv.ListenAndServe())
}
```

- [ ] **Step 8: 运行测试确认通过**

Run: `cd server/submitd && go test ./... && go build -o submitd .`
Expected: `ok submitd`，生成 `submitd` 可执行文件

- [ ] **Step 9: Commit**

```bash
git add server/submitd
git commit -m "feat(submitd): go + sqlite submission service with auth and tests"
```

---

