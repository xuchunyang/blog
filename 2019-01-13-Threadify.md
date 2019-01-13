---
title: M-x threadify
date: Sun, 13 Jan 2019 13:41:50 +0800
tags: elisp
---

Clojure 和 Elixir 用户可能不太喜欢嵌套的函数调用，分别提供 [Threading Macros](https://clojure.org/guides/threading_macros) 和 [Pipe operator](https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2)，比如这样一个 Emacs Lisp 表达式：

```elisp
(reverse (upcase (string-trim " abc ")))
;; => "CBA"
```

改用 Clojure 的 `->`：

```clojure
(-> " abc "
    clojure.string/trim
    clojure.string/upper-case
    clojure.string/reverse)
;; => "CBA"
```

改用 Elixir 的 `|>`：

```elixir
" abc " |> String.trim |> String.upcase |> String.reverse
# => "CBA"
```

昨天我无意中发现这样一个 Vim 插件：

- [marocchino/pipe_converter: A vim plugin that convert your nested functions into pipe operators](https://github.com/marocchino/pipe_converter)

联想到 Emacs Lisp 其实也有 Threading Macro：

1. Emacs 25.1 加入的 `thread-first`
2. 第三方库 `dash.el` 提供的 `->`

所以想给 Emacs Lisp 也实现这样的功能，试了下发现实现非常简单，可能是因为 Emacs Lisp Sexp 是普普通通的 List，无需解析、直接就能用：

```elisp
(defun threadify (exp)
  (cl-labels ((aux (exp acc)
                   (pcase exp
                     (`(,func ,arg1)
                      (aux arg1 (cons func acc)))
                     (`(,func ,arg1 . ,args)
                      (aux arg1 (cons (cons func args) acc)))
                     (X `(-> ,X ,@acc)))))
    (aux exp ())))
```

测试：

```elisp
(threadify '(reverse (upcase (string-trim " abc "))))
;; => (-> " abc " string-trim upcase reverse)

(threadify '(+ (- (* 1 2)) 3 4))
;; => (-> 1 (* 2) - (+ 3 4))
```

为了方便封装成一个命令：

```elisp
(defun threadify-sexp-at-point ()
  (interactive)
  (unless (sexp-at-point)
    (user-error "No sexp at point"))
  (pcase-let ((`(,beg . ,end) (bounds-of-thing-at-point 'sexp)))
    (let ((new (pp-to-string (threadify (sexp-at-point)))))
      (delete-region beg end)
      (insert new))))
```
