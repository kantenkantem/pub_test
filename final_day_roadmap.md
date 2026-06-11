# 【完全版】最終日完成ロードマップ ＆ 全解決指示書 (6月12日版)

ご指摘の通りです。`layout.html` の全体適用が漏れているだけでなく、プロジェクトの現状を精査したところ、**管理者ログインコントローラーが完全に白紙**であったり、**コミットされた `layout.html` 自体にタイポやバインドエラーを引き起こす致命的なバグ**が存在することが判明しました。

これらをすべて解消し、本日中に確実にシステムを完成させるための「完全ロードマップ」を作成しました。

---

## 目次
1. [リーダー（あなた）が朝一に修正・実装すべき最優先コード (AdminController / 共通レイアウト / セッション)](#1-リーダーあなたが朝一に修正実装すべき最優先コード)
2. [【全員共通】共通レイアウト (layout.html) の全画面適用と修正指示](#2-全員共通共通レイアウト-layouthtml-の全画面適用と修正指示)
3. [各メンバー（B〜E）への具体的指示とコピペ用コード集](#3-各メンバーbeへの具体的指示とコピペ用コード集)

---

## 1. リーダー（あなた）が朝一に修正・実装すべき最優先コード

システム全体のログイン・画面遷移、およびデータベースへのプラン保存を正常化させるため、朝一番に以下の3ファイルを修正してください。

### ① 【新規実装】管理者ログインコントローラーの作成
* **ファイル**: [AdminController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AdminController.java)
* **現在の状態**: 1行目のパッケージ宣言以外、中身が**完全に空っぽ（白紙）**です。管理者でログインできず、画面遷移も行えません。
* **実装コード**:
```java
package com.example.demo.controller;

import java.util.Optional;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.Admin;
import com.example.demo.repository.AdminRepository;

@Controller
public class AdminController {

	private final AdminRepository adminRepository;

	public AdminController(AdminRepository adminRepository) {
		this.adminRepository = adminRepository;
	}

	// 管理者ログイン画面表示
	@GetMapping("/admin/login")
	public String showLoginForm() {
		return "admin/login";
	}

	// 管理者ログイン処理 (セッション保存)
	@PostMapping("/admin/login")
	public String login(
			@RequestParam("loginId") String loginId,
			@RequestParam("password") String password,
			HttpSession session,
			Model model) {

		Optional<Admin> adminOpt = adminRepository.findByLoginId(loginId);

		if (adminOpt.isPresent() && adminOpt.get().getPassword().equals(password)) {
			session.setAttribute("loginAdmin", adminOpt.get()); // セッションへ管理者情報を保存
			return "redirect:/admin/accommodations"; // 宿一覧画面へ
		} else {
			model.addAttribute("errorMessage", "管理者IDまたはパスワードが正しくありません。");
			return "admin/login";
		}
	}

	// 管理者ログアウト処理
	@GetMapping("/admin/logout")
	public String logout(HttpSession session) {
		session.removeAttribute("loginAdmin");
		return "redirect:/admin/login";
	}
}
```

### ② 【バグ修正】共通レイアウト (`layout.html`) の修正
* **ファイル**: [layout.html](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/common/layout.html)
* **現在の状態**: 
  1. 2行目が `th:flagment` とタイポしており動作しません（正しくは `th:fragment`）。
  2. CSSのパスが `/src/main/resources/...` となっており、スタイルが一切反映されません（正しくは `/css/style.css`）。
  3. 一般ユーザー用と管理者用のメニューブロックに `th:if` 条件が抜けており、全てのメニューが同時に表示されてしまいます。
* **修正コード**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" th:fragment="layout(content, userType)">

<head>
    <meta charset="UTF-8">
    <title th:text="${userType == 'admin' ? '宿予約システム やどる (管理者モード)' : '宿予約システム やどる'}">宿予約システム やどる</title>
    <!-- 修正: CSSへのリンクパスを適正化 -->
    <link rel="stylesheet" href="/css/style.css">
</head>

<body th:class="${userType == 'admin' ? 'admin-theme' : ''}">

    <header class="layout-header">
        <a href="/" class="header-logo">
            <span>🏨</span>
            <span th:text="${userType == 'admin' ? 'やどる 管理コンソール' : '宿予約 やどる'}">宿予約 やどる</span>
        </a>

        <nav class="layout-nav">
            <!-- 1. 未ログインゲスト用 -->
            <th:block th:if="${userType == 'guest'}">
                <a href="/accommodations">宿を探す</a>
                <a href="/login">ログイン</a>
                <a href="/users/add">新規登録</a>
            </th:block>

            <!-- 2. 一般ユーザーログイン中ナビ (修正: th:ifの追加とログアウトリンクの修正) -->
            <th:block th:if="${userType == 'user'}">
                <span th:if="${session.loginUser != null}" style="font-size: 0.85rem; color: var(--text-muted); margin-right: 0.5rem;">
                    👤 ようこそ、<strong th:text="${session.loginUser.familyName + ' ' + session.loginUser.givenName}">ゲスト</strong> 様
                </span>
                <a href="/accommodations">宿を探す</a>
                <a href="/mypage">マイページ</a>
                <a href="/logout" class="logout-link">ログアウト</a>
            </th:block>

            <!-- 3. 管理者ログイン中ナビ (修正: th:ifの追加) -->
            <th:block th:if="${userType == 'admin'}">
                <span th:if="${session.loginAdmin != null}" style="font-size: 0.85rem; color: var(--text-muted); margin-right: 0.5rem;">
                    🛠️ 担当: <strong th:text="${session.loginAdmin.familyName + ' ' + session.loginAdmin.givenName}">管理者</strong>
                </span>
                <a href="/admin/accommodations">宿管理</a>
                <a href="/admin/reservations">予約照会</a>
                <a href="/admin/users">会員管理</a>
                <a href="/admin/logout" class="logout-link">ログアウト</a>
            </th:block>
        </nav>
    </header>

    <main class="layout-main">
        <div class="content-card">
            <div th:replace="${content}"></div>
        </div>
    </main>

    <footer class="layout-footer">
        <p>&copy; 2026 宿予約システム やどる. All Rights Reserved.</p>
    </footer>
</body>
</html>
```

### ③ 【セッション保存】一般ログイン処理の修正
* **ファイル**: [UserController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/UserController.java)（38行目付近）
* **現在の状態**: ログイン成功時にセッションへユーザー情報を入れていないため、会員用画面にアクセスしても常に未ログイン扱いになります。
* **修正コード**:
```java
	@PostMapping("/login")
	public String login(
			@RequestParam String email,
			@RequestParam String password,
			jakarta.servlet.http.HttpSession session, // セッション引数を追加
			Model model) {

		List<User> userList = userRepository.findByEmailAndPassword(email, password);
		if (userList == null || userList.size() == 0) {
			model.addAttribute("message", "メールアドレス、パスワードが一致しませんでした");
			return "user/login";
		}

		// 修正: セッションへの保存と、マイページへのリダイレクト
		session.setAttribute("loginUser", userList.get(0));
		return "redirect:/mypage";
	}
```

---

## 2. 【全員共通】共通レイアウト (`layout.html`) の全画面適用指示

各メンバーは、自分が作成した個別HTMLファイルの記述を、共通レイアウトに埋め込まれる形式に修正する必要があります。

### 2.1 個別HTML画面へのレイアウト適用方法
個別画面（HTMLファイル）の `<html>` タグに `th:replace` を適用し、画面固有のコンテンツのみを `<body>` 内に記述するように書き換えます。

* **適用パターン（対照表）**:
  個別画面に合わせて、第2引数（`guest`, `user`, `admin`）を出し分けます。

| 適用するユーザー種別 | 適用先HTMLファイル | 記述する `<html>` タグの形 |
| :--- | :--- | :--- |
| **未ログインゲスト (`guest`)** | `user/login.html`<br>`user/register.html`<br>`admin/login.html` | `<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'guest')}">` |
| **一般会員ログイン後 (`user`)** | `user/mypage.html`<br>`user/reservation_detail.html`<br>`accommodation/search.html`<br>`accommodation/detail.html`<br>`reserve/new.html`<br>`reserve/confirm.html` | `<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'user')}">` |
| **管理者 (`admin`)** | `admin/` 配下の全画面 | `<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'admin')}">` |

### 2.2 個別画面の書き換え具体例
メンバーBが担当する `user/login.html` を例にした書き換えのビフォーアフターです。

* **Before (元のコード)**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>ログイン画面</title>
</head>
<body>
    <h2>ログイン画面</h2>
    ...
</body>
</html>
```

* **After (レイアウト適用後)**:
  `head` タグや `title` タグは `layout.html` 側が適用されるため不要になり、`body` タグの中身だけを記述すればよくなります。
```html
<!DOCTYPE html>
<!-- 修正: th:replaceで共通レイアウトを呼び出し、引数に 'guest' を指定 -->
<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'guest')}">
<body>
    <!-- bodyの中身だけが、レイアウト内の <div th:replace="${content}"></div> の位置に挿入されます -->
    <h2>ログイン画面</h2>
    <span th:if="${message}" style="color:red">
        <p th:text="${message}"></p>
    </span>
    <form action="/login" method="post">
        <table border="1">
            <tr>
                <th>メールアドレス</th>
                <td><input type="email" name="email"></td>
            </tr>
            <tr>
                <th>パスワード</th>
                <td><input type="password" name="password"></td>
            </tr>
        </table><br>
        <button>ログイン</button>
    </form><br>
    <a href="/users/add">新規会員登録はこちら</a>
</body>
</html>
```

---

## 3. 各メンバー（B〜E）への具体的指示とコピペ用コード集

### 👥 メンバーBへの指示（一般会員認証・マイページ・退会）
> **指示**: 
> 「ログインに成功した後の『マイページ（`/mypage`）表示処理』と『退会処理（論理削除）』を UserController に追加してください。また、`login.html`、`register.html` は `guest` レイアウト、`mypage.html` は `user` レイアウトを適用するようにHTMLを書き換えてください。」

#### コントローラー側追加コード: [UserController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/UserController.java)
```java
	// 修正: マイページ表示処理の追加
	@GetMapping("/mypage")
	public String mypage(jakarta.servlet.http.HttpSession session, Model model) {
		User loginUser = (User) session.getAttribute("loginUser");
		if (loginUser == null) {
			return "redirect:/login"; // 未ログインはログイン画面へリダイレクト
		}

		User user = userRepository.findById(loginUser.getId()).get();
		model.addAttribute("user", user);

		// メンバーEのReservationRepositoryからユーザーの予約履歴を取得してバインド
		return "user/mypage";
	}

	// 修正: 退会処理（物理削除ではなく、deactivated_atに本日日付をセットする論理削除）
	@PostMapping("/users/deactivate")
	public String deactivate(jakarta.servlet.http.HttpSession session) {
		User loginUser = (User) session.getAttribute("loginUser");
		if (loginUser != null) {
			User user = userRepository.findById(loginUser.getId()).get();
			user.setDeactivatedAt(LocalDate.now()); // 退会日の設定
			userRepository.save(user);
			session.removeAttribute("loginUser"); // セッションクリア
		}
		return "redirect:/login";
	}
```

---

### 👥 メンバーCへの指示（一般会員：宿検索・詳細・予約処理）
> **指示**: 
> 「現在モック状態になっている宿検索（空室・キーワードでの絞り込み）のロジックを実装してください。また、`AccommodationController` や `ReservationController` に関係するHTML（`search.html`, `detail.html`, `new.html`, `confirm.html`）には `user` レイアウトを適用してください。」

#### コントローラー側修正コード: [AccommodationController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AccommodationController.java)
```java
	private final PlanStockRepository planStockRepository; // コンストラクタ引数およびフィールドの追加を忘れずに

	@GetMapping("/accommodations")
	public String show(
			@RequestParam(defaultValue = "") String keyword,
			@RequestParam(required = false) Integer prefectureId,
			@RequestParam(required = false) String checkInDateStr,
			@RequestParam(required = false) String checkOutDateStr,
			@RequestParam(defaultValue = "1") Integer peopleCount,
			@RequestParam(defaultValue = "1") Integer roomCount,
			Model model) {

		model.addAttribute("prefectureList", prefectureRepository.findAll());
		model.addAttribute("acmCategoryList", acmCategoryRepository.findAll());

		// 1. 都道府県名またはキーワードによる曖昧検索
		List<Accommodation> rawHotels;
		if (prefectureId != null && keyword.length() > 0) {
			rawHotels = acmRepository.findByPrefectureIdAndNameContaining(prefectureId, keyword);
		} else if (prefectureId != null) {
			rawHotels = acmRepository.findByPrefectureId(prefectureId);
		} else if (keyword.length() > 0) {
			rawHotels = acmRepository.findByNameContaining(keyword);
		} else {
			rawHotels = acmRepository.findAll();
		}

		List<Accommodation> resultHotels = new ArrayList<>();

		// 2. 在庫および定員に基づく空室フィルタリング (Java側ループ)
		if (checkInDateStr != null && !checkInDateStr.isEmpty() && checkOutDateStr != null && !checkOutDateStr.isEmpty()) {
			LocalDate checkIn = LocalDate.parse(checkInDateStr);
			LocalDate checkOut = LocalDate.parse(checkOutDateStr);
			LocalDate lastNight = checkOut.minusDays(1);
			long stayDays = java.time.temporal.ChronoUnit.DAYS.between(checkIn, checkOut);

			for (Accommodation hotel : rawHotels) {
				List<Plan> rawPlans = planRepository.findByAccommodationId(hotel.getId());
				boolean hotelHasAvailablePlan = false;

				for (Plan plan : rawPlans) {
					if (plan.getPeopleCount() < peopleCount) continue; // 定員不足はスキップ

					List<PlanStock> stocks = planStockRepository.findByPlanIdAndTargetDateBetween(plan.getId(), checkIn, lastNight);
					boolean isAvailable = (stocks.size() == stayDays);

					for (PlanStock stock : stocks) {
						if (stock.getStock() < roomCount) {
							isAvailable = false;
							break;
						}
					}
					if (isAvailable) {
						hotelHasAvailablePlan = true;
						break;
					}
				}
				if (hotelHasAvailablePlan) {
					resultHotels.add(hotel);
				}
			}
		} else {
			resultHotels = rawHotels;
		}

		model.addAttribute("acmList", resultHotels);
		return "accommodation/search";
	}
```

---

### 👥 メンバーDへの指示（管理者：宿・プランCRUD）
> **指示**: 
> 「`AdminAcmController` 内の宿一覧処理において、Modelキー `preName` や `categoryName` をループで上書きする処理を削除してください。また、`AdminPlanController` のプラン追加処理において、DB保存（および90日在庫生成）を実行する `planService` の呼び出しを追加し、HTML側には `admin` レイアウトを適用してください。」

#### 修正①: [AdminAcmController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AdminAcmController.java) 内のループ処理の削除（75〜86行目付近）
* ループ処理は不要です。Thymeleaf側で `th:text="${acm.prefecture.name}"` のように指定すれば、自動でリレーションオブジェクトの名前が参照されます。

---

### 👥 メンバーEへの指示（管理者：予約・会員管理・キャンセル）
> **指示**: 
> 「予約一覧および詳細でDBから正常に値がバインドされるように修正し、完全にコメントアウトされている `AdminUserController` のコメントを解除して会員情報の照会を可能にしてください。また、予約キャンセル後のリダイレクト先の間違いも修正し、すべてのHTMLに `admin` レイアウトを適用してください。」

#### 修正①: [AdminResController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AdminResController.java) (61〜70行目) のバインディング修正
```java
	@GetMapping("/reservations/detail/{resId}")
	public String showResDetail(@PathVariable Integer resId, Model model) {
		// 修正: DBから予約データを取得してバインドする
		Reservation reservation = reservationRepository.findById(resId)
				.orElseThrow(() -> new IllegalArgumentException("Invalid reservation id: " + resId));
		model.addAttribute("reservation", reservation);
		return "admin/reservation_detail";
	}
```

#### 修正②: `AdminResController` のキャンセル後リダイレクト先 (79行目)
* ❌ 修正前: `return "redirect:/reservation_list";`
* ⭕ **修正後**: `return "redirect:/reservations";` （存在する一覧画面のURLに合わせる）

#### 修正③: [AdminUserController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AdminUserController.java) のコメントアウト全解除とコードの適用
* クラス全体のコメントアウトを外し、会員一覧および会員詳細が動作するようにコードを記述してください。
