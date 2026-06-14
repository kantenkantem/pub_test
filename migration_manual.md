# プロジェクト移行マニュアル（team_dev_E_yadoru → team_dev_E_yadoru_fixed）

このマニュアルは、オリジナルリポジトリ `team_dev_E_yadoru` のソースコードに対してバグ修正および機能追加を適用し、完全に動作する `team_dev_E_yadoru_fixed` の状態にするための手順書です。

各ファイルごとの変更内容（Before / After）に従ってソースコードを修正してください。

---

## 📋 修正ファイル一覧と手順

### 1. [ReservationService.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/service/ReservationService.java)
予約登録処理成功時に、新しく生成された予約ID (`reservation.getId()`) をコントローラに返すように戻り値を変更します。

- **修正箇所（L37-L79付近）**:
  - `public boolean processReservation(...)` を `public Integer processReservation(...)` に変更。
  - 在庫切れ等のエラーリターンを `return false;` から `return null;` に変更。
  - 成功時のリターンを `return true;` から `return reservation.getId();` に変更。

```diff
     @Transactional
-    public boolean processReservation(Integer userId, Integer planId, LocalDate checkIn, LocalDate checkOut,
+    public Integer processReservation(Integer userId, Integer planId, LocalDate checkIn, LocalDate checkOut,
             Integer peopleCount, Integer roomCount) {
 
         LocalDate lastNight = checkOut.minusDays(1);
         List<PlanStock> stocks = planStockRepository.findByPlanIdAndTargetDateBetween(planId, checkIn, lastNight);
 
         // 滞在日数分の在庫データがあるか確認
         long stayDays = ChronoUnit.DAYS.between(checkIn, checkOut);
         if (stocks.size() != stayDays) {
-            return false;
+            return null;
         }
 
         // すべての日で部屋数が確保できるかチェック
         for (PlanStock stock : stocks) {
             if (stock.getStock() < roomCount) {
-                return false;
+                return null;
             }
         }
         // 在庫の減算
         for (PlanStock stock : stocks) {
             stock.setStock(stock.getStock() - roomCount);
             planStockRepository.save(stock);
         }
 
         // 予約の登録
         User user = userRepository.findById(userId)
                 .orElseThrow(() -> new IllegalArgumentException("User not found: " + userId));
         Plan plan = planRepository.findById(planId)
                 .orElseThrow(() -> new IllegalArgumentException("Plan not found: " + planId));
 
         Reservation reservation = new Reservation();
         reservation.setUser(user);
         reservation.setPlan(plan);
         reservation.setReservedAt(LocalDateTime.now());
         reservation.setCheckInDate(checkIn);
         reservation.setCheckOutDate(checkOut);
         reservation.setPeopleCount(peopleCount);
         reservation.setRoomCount(roomCount);
         reservation.setIsCancelled(false);
 
         reservationRepository.save(reservation);
-        return true;
+        return reservation.getId();
     }
```

---

### 2. [ReservationController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/ReservationController.java)
空欄日付の型変換エラー対策 (`required = false`化)、セッション情報からの予約処理、および予約完了後の予約詳細へのリダイレクトを行います。

- **修正箇所1 (インポート追加)**:
  `import com.example.demo.entity.User;` および `import jakarta.servlet.http.HttpSession;` を追加します。

- **修正箇所2 (メソッド引数・処理の修正)**:
  - `showPlanDetail`、`confirmRes`、`register` メソッドの引数の `@RequestParam` の `defaultValue = ""` を削除し、`required = false` に変更。
  - `confirmRes` に `HttpSession session` を引数追加し、セッションから `loginUser.id` を取得し `model.addAttribute("userId", ...)` を実行。さらに宿詳細取得バグを `plan.getAccommodation().getId()` を用いるように修正し、宿泊日数に応じた `totalPrice` を計算・追加。
  - `register` (予約登録処理) の `userId` は引数から削除し、セッションから取得するよう変更。予約登録実行後はハードコードされた `resId = 1` ではなく、サービスから返されたIDを用いて予約詳細画面へリダイレクト。

```diff
+import com.example.demo.entity.User;
...
+import jakarta.servlet.http.HttpSession;
...
 	//予約・宿泊プラン詳細画面の表示
 	@GetMapping("/reservations/{planId}")
 	public String showPlanDetail(@PathVariable Integer planId,
-			@RequestParam(defaultValue = "") LocalDate checkInTime,
-			@RequestParam(defaultValue = "") LocalDate checkOutTime,
-			@RequestParam(defaultValue = "") Integer peopleCount,
+			@RequestParam(required = false) LocalDate checkInTime,
+			@RequestParam(required = false) LocalDate checkOutTime,
+			@RequestParam(required = false) Integer peopleCount,
 			Model model) {
...
 	//予約確認画面の表示
 	@GetMapping("/reservations/confirm")
-	public String confirmRes(@RequestParam(defaultValue = "") Integer planId,
-			@RequestParam(defaultValue = "") LocalDate checkInTime,
-			@RequestParam(defaultValue = "") LocalDate checkOutTime,
-			@RequestParam(defaultValue = "") Integer peopleCount,
+	public String confirmRes(@RequestParam(required = false) Integer planId,
+			@RequestParam(required = false) LocalDate checkInTime,
+			@RequestParam(required = false) LocalDate checkOutTime,
+			@RequestParam(required = false) Integer peopleCount,
+			HttpSession session,
 			Model model) {
 
-		model.addAttribute("checkInTime", checkInTime);
-		model.addAttribute("checkOutTime", checkOutTime);
-		model.addAttribute("peopleCount", peopleCount);
-
-		Plan plan = planRepository.findById(planId).get();
-		Accommodation accommodation = acmRepository.findById(planRepository.findById(planId).get().getId()).get();
-		model.addAttribute("plan", plan);
-		model.addAttribute("accommodation", accommodation);
-
-		return "reserve/confirm";
-	}
-
-	//予約登録処理
-	@PostMapping("/reservations/confirm")
-	public String register(
-			@RequestParam(defaultValue = "") Integer userId,
-			@RequestParam(defaultValue = "") Integer planId,
-			@RequestParam(defaultValue = "") LocalDate checkInTime,
-			@RequestParam(defaultValue = "") LocalDate checkOutTime,
-			@RequestParam(defaultValue = "") Integer peopleCount,
-			@RequestParam(defaultValue = "") Integer roomCount) {
-
-		reservationService.processReservation(userId, planId, checkInTime, checkOutTime, peopleCount, roomCount);
-
-		Integer resId = 1;
-
-		return "redirect:/users/" + userId + "/reservations/" + resId;
-	}
+		User loginUser = (User) session.getAttribute("loginUser");
+		if (loginUser == null) {
+			return "redirect:/login";
+		}
+
+		model.addAttribute("userId", loginUser.getId());
+		model.addAttribute("checkInTime", checkInTime);
+		model.addAttribute("checkOutTime", checkOutTime);
+		model.addAttribute("peopleCount", peopleCount);
+
+		Plan plan = planRepository.findById(planId).get();
+		Accommodation accommodation = acmRepository.findById(plan.getAccommodation().getId()).get();
+		model.addAttribute("plan", plan);
+		model.addAttribute("accommodation", accommodation);
+
+		long stayDays = 1;
+		if (checkInTime != null && checkOutTime != null) {
+			stayDays = java.time.temporal.ChronoUnit.DAYS.between(checkInTime, checkOutTime);
+			if (stayDays <= 0) stayDays = 1;
+		}
+		int totalPrice = plan.getTaxIncludePrice() * (int) stayDays;
+		model.addAttribute("totalPrice", totalPrice);
+
+		return "reserve/confirm";
+	}
+
+	//予約登録処理
+	@PostMapping("/reservations/confirm")
+	public String register(
+			@RequestParam(required = false) Integer planId,
+			@RequestParam(required = false) LocalDate checkInTime,
+			@RequestParam(required = false) LocalDate checkOutTime,
+			@RequestParam(required = false) Integer peopleCount,
+			@RequestParam(defaultValue = "1") Integer roomCount,
+			HttpSession session) {
+
+		User loginUser = (User) session.getAttribute("loginUser");
+		if (loginUser == null) {
+			return "redirect:/login";
+		}
+
+		Integer resId = reservationService.processReservation(loginUser.getId(), planId, checkInTime, checkOutTime, peopleCount, roomCount);
+		if (resId == null) {
+			return "redirect:/reservations/" + planId + "?error=stock";
+		}
+
+		return "redirect:/users/" + loginUser.getId() + "/reservations/" + resId;
+	}
+```

---

### 3. [confirm.html (reserve)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/reserve/confirm.html)
ハードコードされた `userId` をセッションからの変数に変更し、合計料金および部屋数を動的表示します。

- **修正箇所（L48〜L65付近）**:
```diff
 		<tr>
 			<th>部屋数</th>
-			<td th:text="a"></td>
+			<td th:text="'1部屋'"></td>
 		</tr>
 		<tr>
 			<th>合計料金</th>
-			<td th:text="a"></td>
+			<td th:text="${totalPrice} + '円'"></td>
 		</tr>
 	</table>
 
 	<form th:action="@{/reservations/confirm}" method="post">
 		<button>確認</button>
-		<input type="hidden" name="userId" th:value="1">
+		<input type="hidden" name="userId" th:value="${userId}">
 		<input type="hidden" name="planId" th:value="${plan.id}">
```

---

### 4. [AccommodationController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AccommodationController.java)
検索フォームの値保持のためのモデル追加、および日付計算順序のバグ（チェックアウトからチェックインを引いていた点）を修正します。

- **修正箇所1 (showメソッド L48-L90付近)**:
  - 各引数を `required = false` に変更。
  - `innDays` 計算を `between(checkInTime, checkOutTime)` に修正。
  - 検索結果を全件取得で上書きしていた `acmList = acmRepository.findAllDto();` を削除。
  - `name`, `prefectureId`, `cateoryId` をモデルに追加。

```diff
 	//宿の検索・一覧画面の表示
 	@GetMapping("/accommodations")
-	public String show(@RequestParam(defaultValue = "") String name,
-			@RequestParam(defaultValue = "") Integer prefectureId,
-			@RequestParam(defaultValue = "") Integer cateoryId,
-			@RequestParam(defaultValue = "") LocalDate checkInTime,
-			@RequestParam(defaultValue = "") LocalDate checkOutTime,
-			@RequestParam(defaultValue = "") Integer peopleCount,
+	public String show(@RequestParam(required = false) String name,
+			@RequestParam(required = false) Integer prefectureId,
+			@RequestParam(required = false) Integer cateoryId,
+			@RequestParam(required = false) LocalDate checkInTime,
+			@RequestParam(required = false) LocalDate checkOutTime,
+			@RequestParam(required = false) Integer peopleCount,
 			Model model) {
 
 		LocalDate reservableDate = LocalDate.now().plusDays(1);
 
 		long innDays = 0;
 		if (checkInTime != null && checkOutTime != null) {
-			innDays = ChronoUnit.DAYS.between(checkOutTime, checkInTime);
+			innDays = ChronoUnit.DAYS.between(checkInTime, checkOutTime);
 		}
...
 		List<AccommodationDto> acmList = acmRepository.findByAllSearch(
 				name,
 				prefectureId,
 				cateoryId,
 				checkInTime,
 				checkOutTime,
 				peopleCount,
 				innDays);
 
-		acmList = acmRepository.findAllDto();
-
 		System.out.println(acmList.size());
 
+		model.addAttribute("name", name);
+		model.addAttribute("prefectureId", prefectureId);
+		model.addAttribute("cateoryId", cateoryId);
 		model.addAttribute("checkInTime", checkInTime);
```

- **修正箇所2 (detailメソッド L93-L98付近)**:
  - 引数を `required = false` に変更。
```diff
 	//宿の詳細画面の表示
 	@GetMapping("/accommodations/{acmId}")
 	public String detail(@PathVariable Integer acmId,
-			@RequestParam(defaultValue = "") LocalDate checkInTime,
-			@RequestParam(defaultValue = "") LocalDate checkOutTime,
-			@RequestParam(defaultValue = "") Integer peopleCount,
+			@RequestParam(required = false) LocalDate checkInTime,
+			@RequestParam(required = false) LocalDate checkOutTime,
+			@RequestParam(required = false) Integer peopleCount,
 			Model model) {
```

---

### 5. [AcmRepository.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/repository/AcmRepository.java)
引数が `null` の場合に絞り込みを行わないよう SQL (JPQL) を動的クエリ形式に修正し、チェックアウト日の在庫カウント不要化のバグを修正します。

- **修正箇所（L51-L68付近）**:
```diff
 			+ "FROM Accommodation acm1 "
 			+ "LEFT JOIN Plan p1 on p1.accommodation.id = acm1.id "
 			+ "LEFT JOIN Prefecture pre1 ON pre1.id = acm1.prefecture.id "
 			+ "LEFT JOIN AcmCategory aca1 on aca1.id = acm1.acmCategory.id "
 			+ "LEFT JOIN PlanStock ps1 on ps1.plan.id = p1.id "
 			+ "WHERE "
-			+ "(SELECT COUNT(*) "
+			+ "(:innDays = 0 OR (SELECT COUNT(*) "
 			+ "FROM PlanStock ps2 "
 			+ "LEFT JOIN Plan p2 ON p2.id = ps2.plan.id "
 			+ "WHERE "
 			+ "p1.id = p2.id "
 			+ "AND COALESCE(ps2.stock , 0) > 0 "
-			+ "AND ps2.targetDate BETWEEN :checkIn AND :checkOut "
-			+ ") > :innDays "
-			+ "AND pre1.id = :prefId "
-			+ "AND aca1.id = :categoryId "
-			+ "AND acm1.name LIKE CONCAT('%', :name , '%') "
-			+ "AND p1.peopleCount >= :peopleCount "
+			+ "AND ps2.targetDate >= :checkIn AND ps2.targetDate < :checkOut "
+			+ ") >= :innDays) "
+			+ "AND (:prefId IS NULL OR pre1.id = :prefId) "
+			+ "AND (:categoryId IS NULL OR aca1.id = :categoryId) "
+			+ "AND (:name IS NULL OR :name = '' OR acm1.name LIKE CONCAT('%', :name , '%')) "
+			+ "AND (:peopleCount IS NULL OR p1.peopleCount >= :peopleCount) "
 			+ "GROUP BY "
```

---

### 6. [search.html (accommodation)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/accommodation/search.html)
検索条件の入力値保持、フォーム部品名のコントローラ連動化、および日付の前後制限JavaScriptの追加を行います。

- **修正箇所1 (form部品名と属性の修正 L26-L66付近)**:
  - `typeOfAcm` -> `cateoryId`、`keyword` -> `name` に name 属性を修正。
  - 各要素に `th:value` もしくは `th:selected` を追加して状態を保持。
  - 日付入力要素に `id="checkInTime"`, `id="checkOutTime"` を追加。

```diff
 	<form th:action="@{/accommodations}" method="get">
 		<select name="prefectureId">
-			<option value="" selected>--- 都道府県 ---</option>
+			<option value="">--- 都道府県 ---</option>
 			<th:block th:each="prefecture : ${prefectureList}" th:object="${prefecture}">
-				<option th:value="*{id}" th:text="*{name}"></option>
-			</th:block>
-		</select>
-
-		<select name="typeOfAcm">
-			<option value="" selected>---宿種別---</option>
+				<option th:value="*{id}" th:text="*{name}" th:selected="*{id} == ${prefectureId}"></option>
+			</th:block>
+		</select>
+
+		<select name="cateoryId">
+			<option value="">---宿種別---</option>
 			<th:block th:each="acmCategory : ${acmCategoryList}" th:object="${acmCategory}">
-				<option th:value="*{id}" th:text="*{name}"></option>
-			</th:block>
-		</select>
-
-		<input type="text" name="keyword" placeholder="宿名">
+				<option th:value="*{id}" th:text="*{name}" th:selected="*{id} == ${cateoryId}"></option>
+			</th:block>
+		</select>
+
+		<input type="text" name="name" th:value="${name}" placeholder="宿名">
 
 		<select name="peopleCount">
-			<option value="" selected>---宿泊人数---</option>
+			<option value="">---宿泊人数---</option>
 			<th:block th:each="peopleNum : ${#numbers.sequence(1,10)}">
-				<option th:value="${peopleNum}" th:text="${peopleNum}"></option>
+				<option th:value="${peopleNum}" th:text="${peopleNum}" th:selected="${peopleNum} == ${peopleCount}"></option>
 			</th:block>
 		</select>
 		名
...
 			<tr>
 				<td>
-					<input type="date" name="checkInTime">
+					<input type="date" id="checkInTime" name="checkInTime" th:value="${checkInTime}">
 				</td>
 				<td>
-					<input type="date" name="checkOutTime">
+					<input type="date" id="checkOutTime" name="checkOutTime" th:value="${checkOutTime}">
 				</td>
 			</tr>
```

- **修正箇所2 (末尾にJavaScriptを追加 L86以降)**:
  `</body>` の直前に、チェックイン日に基づくチェックアウト最小日制御のスクリプトを追記します。
```html
	<script>
		const checkInInput = document.getElementById('checkInTime');
		const checkOutInput = document.getElementById('checkOutTime');

		function updateCheckOutMin() {
			if (checkInInput.value) {
				const checkInDate = new Date(checkInInput.value);
				checkInDate.setDate(checkInDate.getDate() + 1);
				const yyyy = checkInDate.getFullYear();
				const mm = String(checkInDate.getMonth() + 1).padStart(2, '0');
				const dd = String(checkInDate.getDate()).padStart(2, '0');
				const minDate = `${yyyy}-${mm}-${dd}`;
				
				checkOutInput.min = minDate;
				
				if (checkOutInput.value && checkOutInput.value < minDate) {
					checkOutInput.value = minDate;
				}
			} else {
				checkOutInput.removeAttribute('min');
			}
		}

		checkInInput.addEventListener('change', updateCheckOutMin);
		window.addEventListener('DOMContentLoaded', updateCheckOutMin);
	</script>
```

---

### 7. [detail.html (accommodation)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/accommodation/detail.html)
宿詳細画面で、引き継いだ宿泊期間と宿泊人数をタイトル付近に表示します。

- **修正箇所（L7〜L8付近）**:
```diff
 	<h1 th:text="${acm.name}"></h1>
-	<h3 th:text="の宿泊プラン"></h3>
+	<h3>
+		<th:block th:if="${checkInTime != null and checkOutTime != null or peopleCount != null}">
+			（
+			<th:block th:if="${checkInTime != null and checkOutTime != null}">
+				<span th:text="${checkInTime} + ' ～ ' + ${checkOutTime}"></span>
+			</th:block>
+			<th:block th:if="${peopleCount != null}">
+				<span th:if="${checkInTime != null and checkOutTime != null}">、</span>
+				<span th:text="${peopleCount} + '名様'"></span>
+			</th:block>
+			）
+		</th:block>
+		<span>の宿泊プラン</span>
+	</h3>
```

---

### 8. [layout.html (common)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/common/layout.html)
管理者ログイン画面にダークテーマを適用するため、未ログインの管理者状態 (`admin-guest`) に対応します。

- **修正箇所（L12〜L17付近）**:
```diff
-<body th:class="${userType == 'admin' ? 'admin-theme' : ''}">
+<body th:class="${userType == 'admin' || userType == 'admin-guest' ? 'admin-theme' : ''}">
 
     <header class="layout-header">
         <span class="header-logo">
             <span>🏨</span>
-            <span th:text="${userType == 'admin' ? 'やどる 管理コンソール' : '宿予約 やどる'}"></span>
+            <span th:text="${userType == 'admin' || userType == 'admin-guest' ? 'やどる 管理コンソール' : '宿予約 やどる'}"></span>
         </span>
```

---

### 9. [login.html (admin)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/admin/login.html)
レイアウトパラメータを変更してダークテーマを適用し、会員ログインへの相互リンクを追加します。

- **ファイル全体置き換え**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'admin-guest','管理者ログイン')}">

<body>
	<h1>管理者ログイン</h1>
	<span th:if="${message}" style="color:red">
			<p th:text="${message}"></p>
		</span>
	<span><!--エラーメッセージ--></span>
	<form action="/admin/login" method="post">
		<table>
			<tr>
				<td>ログインID</td>
				<td><input type="text" name="loginId"></td>
			</tr>
			<tr>
				<td>パスワード</td>
				<td><input type="password" name="password"></td>
			</tr>
		</table>
		<br>
		<button>ログイン</button>
	</form>
	<br>
	<a href="/login" style="color: var(--primary); font-size: 0.9rem;">一般会員ログインはこちら</a>
</body>

</html>
```

---

### 10. [reservation_detail.html (user)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/user/reservation_detail.html)
レイアウトパスの修正、および空だった画面に詳細表示用テーブルを実装します。

- **ファイル全体置き換え**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'user', '予約詳細')}">

<body>
	<h2>ご予約詳細</h2>
	<a href="/mypage" style="display: inline-block; margin-bottom: 1rem;">マイページへ戻る</a>
	
	<table border="1">
		<tr>
			<th>予約ID</th>
			<td th:text="${reservation.id}"></td>
		</tr>
		<tr>
			<th>宿名</th>
			<td th:text="${reservation.plan.accommodation.name}"></td>
		</tr>
		<tr>
			<th>プラン名</th>
			<td th:text="${reservation.plan.name}"></td>
		</tr>
		<tr>
			<th>チェックイン日</th>
			<td th:text="${reservation.checkInDate}"></td>
		</tr>
		<tr>
			<th>チェックアウト日</th>
			<td th:text="${reservation.checkOutDate}"></td>
		</tr>
		<tr>
			<th>宿泊人数</th>
			<td th:text="${reservation.peopleCount} + '名'"></td>
		</tr>
		<tr>
			<th>部屋数</th>
			<td th:text="${reservation.roomCount} + '部屋'"></td>
		</tr>
		<tr>
			<th>予約状態</th>
			<td th:text="${reservation.isCancelled ? 'キャンセル済' : '予約済'}"></td>
		</tr>
	</table>
</body>

</html>
```

---

### 11. [mypage.html (user)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/user/mypage.html)
履歴の一覧テーブルが予約ごとに別テーブルになってしまう問題を、1つのテーブルでリスト表示するように改善します。

- **ファイル全体置き換え**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'user','マイページ')}">

<body>
	<h2>マイページ（予約履歴）</h2>
	<p th:if="${#lists.isEmpty(res)}">予約はまだありません。</p>
	<table border="1" th:unless="${#lists.isEmpty(res)}">
		<tr>
			<th>詳細</th>
			<th>予約ID</th>
			<th>プラン名</th>
			<th>宿名</th>
			<th>チェックイン日</th>
			<th>チェックアウト日</th>
			<th>状態</th>
		</tr>
		<tr th:each="r : ${res}">
			<td>
				<a th:href="@{/users/{userId}/reservations/{resId}(userId=${r.user.id}, resId=${r.id})}">詳細</a>
			</td>
			<td th:text="${r.id}"></td>
			<td th:text="${r.plan.name}"></td>
			<td th:text="${r.plan.accommodation.name}"></td>
			<td th:text="${r.checkInDate}"></td>
			<td th:text="${r.checkOutDate}"></td>
			<td th:text="${r.isCancelled ? 'キャンセル済' : '予約中'}"></td>
		</tr>
	</table>

</body>

</html>
```

---

### 12. [reservation_detail.html (admin)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/admin/reservation_detail.html)
後半の重複HTMLを削除し、「一覧に戻る」の遷移先パスおよび日時フォーマットを適用します。

- **ファイル全体置き換え**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'admin','予約詳細')}">

<body>
<a href="/admin/reservations">一覧に戻る</a>
<h2>予約詳細画面</h2>
<table>
	<tr>
		<th>予約ID</th>
		<td th:text="${res.id}"></td>
	</tr>
	<tr>
		<th>予約申込日</th>
		<td th:text="${#temporals.format(res.reservedAt, 'yyyy/MM/dd HH:mm')}"></td>
	</tr>
	<tr>
		<th>会員ID</th>
		<td th:text="${res.user.id}"></td>
	</tr>
	<tr>
		<th>会員氏名</th>
		<td th:text="${res.user.name}"></td>
	</tr>
</table>
<table>
	<tr>
		<th>宿名</th>
		<td th:text="${res.plan.accommodation.name}"></td>
	</tr>
	<tr>
		<th>プラン名</th>
		<td th:text="${res.plan.name}"></td>
	</tr>
	<tr>
		<th>チェックイン日</th>
		<td th:text="${res.checkInDate}"></td>
	</tr>
	<tr>
		<th>チェックアウト日</th>
		<td th:text="${res.checkOutDate}"></td>
	</tr>
	<tr>
		<th>予約部屋数</th>
		<td th:text="${res.roomCount}"></td>
	</tr>
	<tr>
		<th>宿泊人数</th>
		<td th:text="${res.peopleCount}"></td>
	</tr>
	<tr>
		<th>合計代金(税込)</th>
		<td th:text="${res.plan.taxIncludePrice}"></td>
	</tr>
</table>
<form th:action="@{/reservations/archive/{resId}(resId=${res.id})}" method="post">
	<button>予約をキャンセルする</button>
</form>

</body>

</html>
```

---

### 13. [reservationList.html (admin)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/admin/reservationList.html)
検索アクションのパス不整合バグを修正し、チェックイン日入力を日付のみ (`date`) に変更、値の保持 (`th:value`)、および申込日の日時表示フォーマットの適用を行います。

- **ファイル全体置き換え**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'admin','予約一覧')}">

<body>
	<aside>
		<h2>検索</h2>
		<form action="/admin/reservations" method="get">
			会員名
			<input type="text" name="name">
			宿名
			<input type="text" name="acmName">
			チェックイン日
			<input type="date" name="checkInDate" th:value="${checkInDate}">
			<button>検索</button>
		</form>
	</aside>
	<main>
		<h2>予約一覧</h2>
		<table border="1">
			<tr>
				<th></th>
				<th>予約ID</th>
				<th>会員名</th>
				<th>宿名</th>
				<th>プラン名</th>
				<th>宿泊日</th>
				<th>予約申込日</th>
				<th>予約状況</th>
			</tr>
			<tr th:each="reservation : ${reservations}" th:object="${reservation}">
				<td>
					<a th:href="@{/reservations/detail/{resId}(resId=*{id})}">詳細</a>
				</td>
				<td th:text="*{id}"></td>
				<td th:text="*{user.name}"></td>
				<td th:text="*{plan.accommodation.name}"></td>
				<td th:text="*{plan.name}"></td>
				<td th:text="*{checkInDate}"></td>
				<td th:text="${#temporals.format(reservation.reservedAt, 'yyyy/MM/dd HH:mm')}"></td>
				<td th:text="*{isCancelled ? 'キャンセル済' : '予約済'}"></td>
				
			</tr>
		</table>
	</main>
</body>

</html>
```

---

### 14. [AdminResController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AdminResController.java)
管理側の予約検索のチェックイン日パラメータを `LocalDate` 型として受け取るようにシグネチャを修正します。

- **修正箇所1 (インポート追加)**:
  `import java.time.LocalDate;` を追加。
- **修正箇所2 (listResメソッド L33-L55付近)**:
  `@RequestParam(required=false) Integer date` から `@RequestParam(required=false) LocalDate checkInDate` に変更し、モデルに `checkInDate` を追加。

```diff
+import java.time.LocalDate;
...
 	//予約一覧画面の表示
 	@GetMapping("/admin/reservations")
 	public String listRes(
 			@RequestParam(defaultValue="")String keyword,
-			@RequestParam(required=false)Integer date,
+			@RequestParam(required=false)LocalDate checkInDate,
 			@RequestParam(defaultValue="")String fn,
 			@RequestParam(defaultValue="")String gn,
 			Model model) {
 	
 		//予約一覧情報の取得
 		List<Reservation> reservationList=reservationRepository.findAll();
 		
         //予約の絞り込みと検索結果の表示
 		List<User> userList=userRepository.findAll();
 		if(keyword.length()>0) {
 			userList=userRepository.findByFamilyNameContainingOrGivenNameContaining(keyword,keyword);
 		}
 		model.addAttribute("keyword",keyword);
+		model.addAttribute("checkInDate",checkInDate);
 		model.addAttribute("users",userList);
 		model.addAttribute("reservations",reservationList);
```

---

### 15. [user_detail.html (admin)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/admin/user_detail.html)
モデルに存在しない直接参照変数 `${postalCode}` などを、`${user.postalCode}` などの正しいオブジェクト経由の記述に修正し、ヘッダーの名前を正しく表示させます。

- **修正箇所（L7〜L43付近）**:
```diff
-<h2>[[${<!--会員名-->}]]様</h2>
+<h2>[[${user.name}]]様</h2>
 <table>
 	<tr>
 		<th>会員ID</th>
 		<td th:text="${user.id}"></td>
 	</tr>
 	<tr>
 		<th>名前</th>
 		<td th:text="${user.name}"></td>
 	</tr>
 	<tr>
 		<th>メールアドレス</th>
 		<td th:text="${user.email}"></td>
 	</tr>
 </table>
 <table>
 	<tr>
 		<th>郵便番号</th>
-		<td th:text="${postalCode}"></td>
+		<td th:text="${user.postalCode}"></td>
 	</tr>
 	<tr>
 		<th>住所</th>
-		<td th:text="${address}"></td>
+		<td th:text="${user.address}"></td>
 	</tr>
 	<tr>
 		<th>電話番号</th>
-		<td th:text="${phoneNumber}"></td>
+		<td th:text="${user.phoneNumber}"></td>
 	</tr>
 	<tr>
 		<th>生年月日</th>
-		<td th:text="${birthDate}"></td>
+		<td th:text="${user.birthDate}"></td>
 	</tr>
 	<tr>
 		<th>登録日</th>
-		<td th:text="${registeredAt}"></td>
+		<td th:text="${user.registeredAt}"></td>
 	</tr>
 </table>
```

---

### 16. [acm_detail.html (admin)](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/templates/admin/acm_detail.html)
宿カテゴリ部分のHTMLタグ整合性エラーを修正し、各プランごとに別テーブルで表示されていたものを1つの綺麗なテーブルとして並ぶよう統合します。

- **ファイル全体置き換え**:
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" th:replace="~{common/layout :: layout(~{::body}, 'admin','詳細：宿泊施設')}">

<head>
	<meta charset="UTF-8">
	<title>宿詳細情報</title>
</head>

<body>
	<a href="/admin/accommodations">一覧に戻る</a>
	<main>
		<h1 th:text="${acm.name}"></h1>
		<a th:href="@{/admin/accommodations/edit/{acmId}(acmId=${acm.id})}">編集</a>
		<table>
			<tr>
				<th>郵便番号</th>
				<td th:text="${acm.postalCode}"></td>
			</tr>
			<tr>
				<th>住所</th>
				<td th:text="${preName.name}+${acm.address}"></td>
			</tr>
			<tr>
				<th>電話番号</th>
				<td th:text="${acm.phoneNumber}"></td>
			</tr>
			<tr>
				<th>チェックイン時間</th>
				<td th:text="${acm.checkInTime}"></td>
			</tr>
			<tr>
				<th>チェックアウト時間</th>
				<td th:text="${acm.checkOutTime}"></td>
			</tr>
			<tr>
				<th>宿カテゴリー</th>
				<td th:text="${acm.acmCategory.name}"></td>
			</tr>
			<tr>
				<th>プラン数</th>
				<td th:text="${planCount}"></td>
			</tr>
		</table>
	</main>
	<p></p>
	<aside>
		<form th:action="@{/plans/add/{acmId}(acmId=${acm.id})}" method="get">
			<button>宿泊プランを追加する</button>
		</form>
		<th:block th:if="${not #lists.isEmpty(planList)}">
			<table>
				<tr>
					<th>プラン名</th>
					<th>プラン内容</th>
					<th>料金(税込)</th>
					<th>部屋数</th>
					<th>定員人数</th>
					<th>朝食</th>
				</tr>
				<tr th:each="plan : ${planList}">
					<td>
						<a th:href="@{/plans/detail/{planId}(planId=${plan.id})}" th:text="${plan.name}"></a>
					</td>
					<td th:text="${plan.content}"></td>
					<td th:text="${plan.taxIncludePrice} + '円'"></td>
					<td th:text="${plan.roomCount} + '部屋'"></td>
					<td th:text="${plan.peopleCount} + '人'"></td>
					<td th:text="${plan.isBreakfast ? '〇' : '×'}"></td>
				</tr>
			</table>
		</th:block>
		<p th:if="${#lists.isEmpty(planList)}">登録されている宿泊プランはありません。</p>
	</aside>

</body>

</html>
```

---

### 17. [style.css](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/static/css/style.css)
テーブル、各種入力要素 (input, select, textarea) およびボタン要素に対する、モダンで高品質な共通スタイルを末尾に追記します。

- **追記内容 (ファイルの最下部に追加)**:
```css
/* ==========================================
   テーブル (Table) スタイル
   ========================================== */
table {
    width: 100%;
    border-collapse: collapse;
    margin: 1.5rem 0;
    font-size: 0.95rem;
    text-align: left;
    background-color: var(--card-bg);
    border: 1px solid var(--border-color);
    border-radius: 0.5rem;
    overflow: hidden;
}

table th, table td {
    padding: 0.75rem 1rem;
    border: 1px solid var(--border-color);
}

table th {
    background-color: rgba(79, 70, 229, 0.05);
    color: var(--text-main);
    font-weight: 600;
}

.admin-theme table th {
    background-color: rgba(56, 189, 248, 0.08);
}

table tr:hover {
    background-color: rgba(79, 70, 229, 0.02);
}

.admin-theme table tr:hover {
    background-color: rgba(56, 189, 248, 0.03);
}

/* ==========================================
   フォーム (Form) ・ 入力要素 スタイル
   ========================================== */
form {
    margin: 1.5rem 0;
}

label {
    display: block;
    margin-bottom: 0.5rem;
    font-weight: 500;
    color: var(--text-main);
}

input[type="text"],
input[type="password"],
input[type="email"],
input[type="date"],
input[type="datetime-local"],
select,
textarea {
    width: 100%;
    padding: 0.625rem 0.875rem;
    font-size: 0.95rem;
    border: 1px solid var(--border-color);
    border-radius: 0.375rem;
    background-color: var(--card-bg);
    color: var(--text-main);
    box-sizing: border-box;
    transition: border-color 0.2s, box-shadow 0.2s;
}

input[type="text"]:focus,
input[type="password"]:focus,
input[type="email"]:focus,
input[type="date"]:focus,
input[type="datetime-local"]:focus,
select:focus,
textarea:focus {
    outline: none;
    border-color: var(--primary);
    box-shadow: 0 0 0 3px rgba(79, 70, 229, 0.15);
}

.admin-theme input[type="text"]:focus,
.admin-theme input[type="password"]:focus,
.admin-theme input[type="email"]:focus,
.admin-theme input[type="date"]:focus,
.admin-theme input[type="datetime-local"]:focus,
.admin-theme select:focus,
.admin-theme textarea:focus {
    box-shadow: 0 0 0 3px rgba(56, 189, 248, 0.2);
}

/* フォームテーブル内の入力フィールド調整 */
table td input,
table td select {
    width: 100%;
}

/* ==========================================
   ボタン (Button) スタイル
   ========================================== */
button, 
.btn {
    display: inline-block;
    padding: 0.625rem 1.25rem;
    font-size: 0.95rem;
    font-weight: 600;
    color: #ffffff;
    background-color: var(--primary);
    border: none;
    border-radius: 0.375rem;
    cursor: pointer;
    transition: background-color 0.2s, transform 0.1s;
    text-decoration: none;
    text-align: center;
}

button:hover,
.btn:hover {
    background-color: var(--primary-hover);
}

button:active,
.btn:active {
    transform: scale(0.98);
}

/* キャンセルボタン用 */
button.danger,
.btn-danger {
    background-color: #ef4444;
}

button.danger:hover,
.btn-danger:hover {
    background-color: #dc2626;
}
```

---

### 18. [data.sql](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/resources/data.sql)
未来のテスト日程における在庫データ (`plan_stocks`) を追加します。

- **修正箇所 (L103-L108付近)**:
```diff
 INSERT INTO plan_stocks(plan_id, target_date, stock)
 VALUES
 	(1, '2026-06-06', 7),
 	(1, '2026-06-07', 7),
-	(2, '2026-06-06', 10)
+	(1, '2026-06-20', 7),
+	(1, '2026-06-21', 7),
+	(2, '2026-06-06', 10),
+	(2, '2026-06-20', 10),
+	(2, '2026-06-21', 10)
 ;
```
