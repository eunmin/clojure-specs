# Specs

Programming Clojure(https://pragprog.com/book/shcloj3/programming-clojure-third-edition) 5장을 정리했습니다.

- Clojure는 다이나믹 타입 언어라서 기본 타입으로로 데이터를 표현한다.
- 다이나믹 타입의 장점들이 있지만 코드가 명확하지 않다는 단점도 있다.
- 다음 코드 중 clojure 코드는 `ingredient` 인자의 타입이 map이기 때문에 자바 코드에 비해 명확하지 않다.
  그래서 데이터 타입이 어떤지 알기 위해 문서나 코드를 봐야 한다.

```java
public class Ingredient {
  private String name;
  private double quantity;
  private Unit unit;
  // ...
  public static Ingredient scale(Ingredient ingredient, double factor) {
    ingredient.setQuantity(ingredient.getQuantity() * factor);
    return ingredient;
  }
}
```

```clojure
(defn scale-ingredient [ingredient factor]
  (update ingredient :quantity * factor))
```

- clojure 1.9에서 데이터를 서술하기 위해 `specs` 라이브러리가 생겼다.
- 위 예제를 specs 라이브러리를 써서 데이터를 서술해 봤다. (alias `s`는 `clojure.spec.alpha`)

```clojure
;; Specs describing an ingredient
(s/def ::ingredient (s/keys :req [::name ::quantity ::unit]))
(s/def ::name string?)
(s/def ::quantity number?)
(s/def ::unit keyword?)

;; Function spec for scale-ingredient
(s/fdef scale-ingredient
  :args (s/cat :ingredient ::ingredient :factor number?)
  :ret  ::ingredient)
```

## Specs 정의하기

- 보통 `clojure.spec.alpha` 네임스페이스를 `s`로 alias 걸어 사용한다.

```clojure
(require '[clojure.spec.alpha :as s])
```

- `s/def` 매크로로 전역 specs를 등록 할 수 있다.

```clojure
(s/def name spec)
```

- specs 이름은 qualified 키워드를 쓴다. 사용할 때는 alias를 걸고 auto-resolved 키워드(`::`)를
  사용하면 편하다.

```clojure
(s/def :my.app/company-name string?)
```

- 보통 specs는 각각의 함수 위에 정의하거나 네임스페이스를 따로 만들어 모아둔다. clojure.core 함수 스팩은
  `clojure.core.specs.alpha` 네임스페이스에 있다.

## 데이터 체크하기

- specs는 함수에 예상하지 못한 데이터가 적용되는 것을 막을 수 있다. 이유도 알 수 있다.

### Predicates

- Predicates는 값을 하나 받아서 specs에 맞는지 판단해 ture 또는 false를 내주는 함수다.
- clojure에 기본으로 있는 Predicates 함수(string? boolean? keyword? ...)로 값 하나에 대한 체크를 할 수 있다.
- 아래는 string? Predicates 함수로 되어 있는 간단한 specs를 체크하는 코드다.

```clojure
(s/def :my.app/company-name string?)

(s/valid? :my.app/company-name "Acme Moving")
;; => true
(s/valid? :my.app/company-name 100)
;; => false
```

### Enumerated values

- set은 자체가 함수기 때문에 아래 처럼 체크 할 수 있다.

```clojure
(s/def :marble/color #{:red :green :blue})

(s/valid? :marble/color :red)
;; => true
(s/valid? :marble/color :pink)
;; => false
```

- 아래도 비슷한 코드지만 더 좋은 방법이 있다.

```clojure
(s/def ::bowling/roll #{0 1 2 3 4 5 6 7 8 9 10})

(s/valid? ::bowling/roll 5)
;;=> true
```

### Range Specs

- `s/int-in`, `s/double-in`, `s/inst-in` 처럼 구간이 있는 값을 체크하는 함수를 쓰면 더 좋다.

```clojure
(s/def ::bowling/ranged-roll (s/int-in 0 11))

(s/valid? ::bowling/ranged-roll 10)
;;=> true
```

### nil 다루기

- `nil`을 허용하는 specs를 만드려면 `s/nilable`를 사용하는 것이 `s/or`로 조합하는 것보다 더 빠르다.

```clojure
(s/def ::my.app/company-name-2 (s/nilable string?))

(s/valid? ::my.app/company-name-2 nil)
;;=> true
```

### Logical Specs

- `s/and`나 `s/or`로 Predicates 함수를 조합해 specs를 만들 수 있다.

```clojure
(s/def ::odd-int (s/and int? odd?))

(s/valid? ::odd-int 5)
;; => true

(s/valid? ::odd-int 10)
;; => false

(s/valid? ::odd-int 5.2)
;; => false
```

- `s/or`에 매칭 된 Predicates와 값을 받기 위해 아래 코드 처럼 쓸 수 있다. 매칭 결과는 MapEntry 형태다.

```clojure
(s/def ::odd-or-42 (s/or :odd ::odd-int :42 #{42}))

(s/conform ::odd-or-42 42)
;; => [:42 42]
(s/conform ::odd-or-42 19)
;; => [:odd 19]
```

- `s/conform`은 매칭된 결과를 리턴하지만 `s/explain`는 매칭되지 않은 이유를 알려준다. `0` 값은
  `:odd` 키의 `:user/odd-int` specs에서 `odd?`에 만족하지 못했고 `:42` 키에 `:user/odd-or-42`
  specs에서 `#{42}`에도 만족하지 못해 체크에 실패 했다.

```clojure
(s/explain ::odd-or-42 0)
| val: 0 fails spec: :user/odd-int at: [:odd] predicate: odd?
| val: 0 fails spec: :user/odd-or-42 at: [:42] predicate: #{42}
```

- `s/explain`은 `console`로 출력되지만 `s/explain-str`나 `s/explain-data`를 쓰면 리턴 값으로
  받을 수 있다.

### 컬렉션 Specs

- `s/coll-of`는 리스트, 벡터, 셑, `seqs` 같은 데이터에 사용한다.

```clojure
(s/def ::names (s/coll-of string?))

(s/valid? ::names ["Alex" "Stu"])
;; => true

(s/valid? ::names #{"Alex" "Stu"})
;; => true

(s/valid? ::names '("Alex" "Stu"))
;; => true
```

- `s/coll-of`는 다음과 같은 옵션이 있다.
  - `:kind`는 `vector?`나 `set?` 같은 컬렉션의 종류를 지정하는데 사용한다.
  - `:into`는 `conform` 결과를 담을 컬렉션 타입으로 `[]`, `()`, `#{}`를 지정한다.
  - `:count`는 컬렉션의 아이템 수를 지정하는데 사용한다.
  - `:min-count`는 컬렉션의 최소 아이템 수를 지정하는데 사용한다.
  - `:max-count`는 컬렉션의 최대 아이템 수를 지정하는데 사용한다.
  - `:distinct`값이 `true`면 컬렉션 값이 유일해야한다.

- 아래는 예제 코드다.

```clojure
(s/def ::my-set (s/coll-of int? :kind set? :min-count 2))

(s/valid? ::my-set #{10 20})
;; => true
```

- `s/coll-of`와 비슷하게 데이터가 맵인 경우 `s/map-of`를 사용한다.

```clojure
(s/def ::scores (s/map-of string? int?))

(s/valid? ::scores {"Stu" 100, "Alex" 200})
;; => true
```

- ??? `s/map-of`에서 키는 기본적으로 `conform`하지 않는다. 만약 쓰려면 `:conform-keys` 옵션을 쓴다.

### 컬렉션 샘플링

- 큰 컬렉션인 경우 모든 값을 체크하지 않을 수 도 있는데 이 경우에 `s/every`, `s/every-kv` 쓸 수 있다.
  사용법은 `s/coll-of`와 `s/map-of`와 같고 `s/*coll-check-limit*` 개수까지 만 체크한다. (기본 값은 101)

### 튜플

- 튜플은 고정값의 벡터로 아래 처럼 specs를 만들 수 있다.

```clojure
(s/def ::point (s/tuple float? float?))

(s/conform ::point [1.3 2.7])
;; => [1.3 2.7]
```

### Information 맵

- clojure는 도메인 객체를 아래 처럼 well-known 필드 맵으로 만든다.

```clojure
{::music/id #uuid "40e30dc1-55ac-33e1-85d3-1f1508140bfc"
 ::music/artist "Rush"
 ::music/title "Moving Pictures"
 ::music/date #inst "1981-02-12"}
```

- 각 필드의 specs는 이렇게 정의 할 수 있다.

```clojure
(s/def ::music/id uuid?)
(s/def ::music/artist string?)
(s/def ::music/title string?)
(s/def ::music/date inst?)
```

- `s/keys`로 맵 specs을 정의 할 수 있다.

```clojure
(s/def ::music/release
  (s/keys :req [::music/id]
          :opt [::music/artist
                ::music/title
                ::music/date]))
```

- unqualified 키를 지원하려면 `:req-un`과 `:opt-un`를 쓰면 된다.

```clojure
{:id #uuid "40e30dc1-55ac-33e1-85d3-1f1508140bfc"
 :artist "Rush"
 :title "Moving Pictures"
 :date #inst "1981-02-12"}

(s/def ::music/release-unqualified
 (s/keys :req-un [::music/id]
         :opt-un [::music/artist
                  ::music/title
                  ::music/date]))
```

## 함수 체크하기

- 함수 specs는 함수의 인자를 정의하는 `args`, 리턴 값을 정의하는 `ret`, 인자와 리턴 값의 관계를 정의하는
  `fn`으로 정의 할 수 있다.
- `args`는 컬렉션에 대한 정의지만 regex op specs을 사용해서 더 정교하게 정의 할 수 있다.

### Sequences With Structure

- `s/cat`에서 regex op spec을 많이 사용한다. 아래는 `s/cat`으로 컬렉션을 정의하는 단순한 코드다.

```clojure
(s/def ::cat-example (s/cat :s string? :i int?))

(s/valid? ::cat-example ["abc" 100])
;; => true
```

- Predicates 앞에 있는 키워드는 `conform`에 결과를 담는 키워드다.

```clojure
(s/conform ::cat-example ["abc" 100])
;; => {:s "abc", :i 100}
```

- 다음은 `s/alt`로 regex op specs를 사용하는 코드다.

```clojure
(s/def ::alt-example (s/alt :i int? :k keyword?))

(s/valid? ::alt-example [100])
;; => true

(s/valid? ::alt-example [:foo])
;; => true

(s/conform ::alt-example [:foo])
;; => [:k :foo]
```

#### Repetition Operators

- 반복 연산자는 3가지가 있다. `s/?`는 0 또는 1, `s/*`는 0 이상, `s/+` 1 이상을 나타낸다.

```clojure
(s/def ::oe (s/cat :odds (s/+ odd?) :even (s/? even?)))

(s/conform ::oe [1 3 5 100])
;; => {:odds [1 3 5], :even 100}
```

- `:odds`에는 `s/+` `odd?` 조건에 따라 `[1 3 5]`가 `:even`에는 `s/?` `even?` 조건에 따라 100이
  매칭 됬다.

- 각각의 조건은 이름을 붙일 수 도 있다.

```clojure
(s/def ::odds (s/+ odd?))
;; => :user/odds

(s/def ::optional-even (s/? even?))
;; => :user/optional-even

(s/def ::oe2 (s/cat :odds ::odds :even ::optional-even))
;; => :user/oe2

(s/conform ::oe2 [1 3 5 100])
;; => {:odds [1 3 5], :even 100}
```

#### Variable Argument Lists

- 가변 인자 함수인 경우 `s/*`을 사용할 수 있다. 0 이상의 인자를 갖는 함수는 아래 코드 처럼 정의 할 수 있다.

```clojure
(s/def ::println-args (s/* any?))
```

- `clojure.set/intersection` 같은 경우는 아래 같은 인자를 갖는다.

```clojure
(doc clojure.set/intersection)
| -------------------------
| clojure.set/intersection
| ([s1] [s1 s2] [s1 s2 & sets])
|   Return a set that is the intersection of the input sets
-> nil
```

- 다음 코드 처럼 specs를 정의할 수 있지만

```clojure
(s/def ::intersection-args
  (s/cat :s1 set?
         :sets (s/* set?)))

(s/conform ::intersection-args '[#{1 2} #{2 3} #{2 5}])
;; {:s1 #{1 2}, :sets [#{3 2} #{2 5}]}
```

- 모든 타입이 `set?`이기 때문에 `s/+`로 정의하는 것이 간단하다.

```clojure
(s/def ::intersection-args-2 (s/+ set?))

(s/conform ::intersection-args-2 '[#{1 2} #{2 3} #{2 5}])
;; => [#{1 2} #{3 2} #{2 5}]
```

- `atom` 같은 경우 `(atom x & options)` 처럼 `options` 맵을 갖는데 이런 경우 `s/key*`로
  정의할 수 있다.

```clojure
(s/def ::meta map?)

(s/def ::validator ifn?)

(s/def ::atom-args
  (s/cat :x any? :options (s/keys* :opt-un [::meta ::validator])))

(s/conform ::atom-args [100 :meta {:foo 1} :validator int?])
;; => {:x 100,
;;     :options {:meta {:foo 1},
;;     :validator #object[clojure.core$int_QMARK_ ...]}}
```

#### Multi-arity Argument Lists

- `repeat`는 `[x]` 또는 `[n x]` 인자를 갖는데 아래 처럼 specs를 정의 할 수 있다.

```clojure
(doc repeat)
| -------------------------
| clojure.core/repeat
| ([x] [n x])
| Returns a lazy (infinite!, or length n if supplied) sequence of xs. -> nil

(s/def ::repeat-args
  (s/cat :n (s/? int?) :x any?))

(s/conform ::repeat-args [100 "foo"])
;; => {:n 100, :x "foo"}

(s/conform ::repeat-args ["foo"])
;; => {:x "foo"}
```

### Specifying Functions

- 함수 값에 specs를 정의하는 방법을 `rand` 함수 예로 알아보자.

```clojure
clojure.core/rand
([] [n])
  Returns a random floating point number between 0 (inclusive) and
  n (default 1) (exclusive).

;; 인자는 숫자를 받거나 아무것도 받지 않을 수 있다.
(s/def ::rand-args (s/cat :n (s/? number?)))

;; 리턴 값은 더블 값이다.
(s/def ::rand-ret double?)

;; 인자와 리턴 값의 관계는 함수로 정의하는데 함수는 conform된 args, ret를 인자로 넘겨주고 true/false 값을 리턴한다.
(s/def ::rand-fn
  (fn [{:keys [args ret]}]
    (let [n (or (:n args) 1)] (cond (zero? n) (zero? ret)
                (pos? n) (and (>= ret 0) (< ret n))
                (neg? n) (and (<= ret 0) (> ret n))))))

(s/fdef clojure.core/rand
  :args ::rand-args
  :ret  ::rand-ret
  :fn   ::rand-fn)
```

### 익명 함수

- 익명 함수는 `s/fspec`으로 정의하고 사용 법은 `s/fdef`와 같다. 아래는 함수를 리턴하는 `opposite` 함수의
  정의 코드다.

```clojure
(defn opposite [pred]
  (comp not pred))

(s/def ::pred
  (s/fspec :args (s/cat :x any?)
           :ret  boolean?))
(s/fdef opposite
  :args (s/cat :pred ::pred)
  :ret ::pred)
```

### Instrumenting Functions

- 개발이나 테스트를 할 때 함수에 들어온 인자 값을 확인하기 위해 instrumentation (stest/instrument)을
  사용할 수 있다. 프로덕션 환경에 사용하기는 적합하지 않다.

- Instrumentation의 목적은 구현이 잘 되었는지의 여부가 아닌 실행이 잘될 것인지 체크하는데 목적이 있기 때문에
  `:ret`나 `:fn`은 체크하지 않고 `:args`만 체크한다.

- `stest/instrument`를 부르면 사용 할 수 있다. `stest/enumerate-namespace`로 네임스페이스에 specs가
  정의돈 모든 함수를 instrument 할 수 도 있다.

```clojure
(require '[clojure.spec.test.alpha :as stest])

(stest/instrument 'clojure.core/rand)

(stest/instrument (stest/enumerate-namespace 'clojure.core))
```

- 아래는 specs에 맞지 않게 부른 경우 나오는 에러 메시지다.

```clojure
(rand :boom)
| ExceptionInfo Call to #'clojure.core/rand did not conform to spec: | In: [0] val: :boom fails spec: :user/rand-args
| at: [:args :n] predicate: number?
| :clojure.spec.alpha/args (:boom)
| :clojure.spec.alpha/failure :instrument
| :clojure.spec.test.alpha/caller {...}
```

## 제너레이티브 함수 테스트

- `stest/check`로 제너레이티브 테스트를 할 수 있다. `stest` alias는 `clojure.spec.test.alpha`다.
- specs 제너레이터를 사용하려면 `test.check` 라이브러리를 추가해야한다. 보통 `dev`나 `test`환경에 추가한다.

```clojure
;; leiningen
:profiles {:dev {:dependencies [[org.clojure/test.check "0.10.0-alpha2"]]}}
```

```clojure
;; boot
(set-env!
 :dependencies '[[org.clojure/test.check "0.10.0-alpha2" :scope "test"]])
```

- `check`를 실행하면 `:args`에 맞는 1000개의 데이터가 생성된다.
- 아래 코드는 `clojure.core/symbol` 함수 specs를 만들고 제너레이티브 테스트를 하는 예제다.

```clojure
(doc symbol)
| -------------------------
| clojure.core/symbol
| ([name] [ns name])
|   Returns a Symbol with the given namespace and name.

(s/fdef clojure.core/symbol
  :args (s/cat :ns (s/? string?) :name string?) :ret symbol?
  :fn (fn [{:keys [args ret]}]
          (and (= (name ret) (:name args))
            (= (namespace ret) (:ns args)))))

(stest/check 'clojure.core/symbol)
;; => ({:sym clojure.core/symbol
;;      :spec #object[clojure.spec.alpha$fspec_impl$reify__14282 ...],
;;      :clojure.spec.test.check/ret {
;;        :result true,
;;        :num-tests 1000,
;;        :seed 1485241441400}})
```

### 예제

- `s/exercise`로 제너레이티브 테스트에서 사용하는 샘플 데이터를 만들어 볼 수 있다.

```clojure
(s/exercise (s/cat :ns (s/? string?) :name string?))
;; => ([("" "") {:ns "", :name ""}]
;;     [("F" "") {:ns "F", :name ""}]
;;     [("s" "73") {:ns "s", :name "73"}]
;;     [("u") {:name "u"}]
;;     [("") {:name ""}]
;;     [("" "3y") {:ns "", :name "3y"}]
;;     [("t" "9pudu") {:ns "t", :name "9pudu"}]
;;     [("Xhw25" "nPR7C9C") {:ns "Xhw25", :name "nPR7C9C"}] [("FXs3E" "N") {:ns "FXs3E", :name "N"}] [("UhUN5dZK1" "le8") {:ns "UhUN5dZK1", :name "le8"}])
```

#### s/and와 제너레이터 조합하기

- 어떤 경우는 정의가 명확하지 않아 샘플 데이터를 자동으로 만들 수 없는데 다음 예를 보자.

```clojure
(defn big? [x] (> x 100))
(s/def ::big-odd (s/and odd? big?))

(s/exercise ::big-odd)
;; => Unable to construct gen at: [] for: odd?
```

- 이 경우 첫번째 컴포넌트에 타입을 지정해서 해결 할 수 있다.

```clojure
(s/def ::big-odd-int (s/and int? odd? big?))

(s/exercise ::big-odd-int)
;; => ([125 125] [177 177] [22547 22547] [6165 6165] [915 915] [303 303] [3789257 3789257] [139 139] [935 935] [5571 5571])
```

#### 커스텀 제너레이터 만들기

- `clojure.spec.gen.alpha` 네임스페이스의 `gen`과 `s/with-gen`으로 커스텀 제너레이트를 만들 수 있다.

```clojure
(require '[clojure.spec.gen.alpha :as gen])
(require '[clojure.string :as str])

(s/def ::sku
  (s/with-gen (s/and string? #(str/starts-with? % "SKU-"))
    (fn [] (gen/fmap #(str "SKU-" %) (s/gen string?)))))

(s/exercise ::sku)
;; => (["SKU-" "SKU-"] ["SKU-" "SKU-"] ["SKU-Ep" "SKU-Ep"] ["SKU-v38" "SKU-v38"] ["SKU-Re7s" "SKU-Re7s"] ["SKU-pIVb" "SKU-pIVb"] ["SKU-f2MB0" "SKU-f2MB0"] ["SKU-C9VWD" "SKU-C9VWD"] ["SKU-j99Q" "SKU-j99Q"] ["SKU-mP" "SKU-mP"])
```

- `gen/fmap`으로 `SKU-`로 시작하고 뒤에는 `string?` 제너레이터의 결과를 붙이는 커스텀 제너레이터를 만들어 ::sku
   specs에 적용했다.
