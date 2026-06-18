# 修正内容ウォークスルー（手動適用用）

> [!IMPORTANT]
> すべてのファイルは **手動で** 変更してください。各セクションに「変更前→変更後」の差分を示します。

---

## 1. ReservationRepository.java

**対象ファイル:** `src/main/java/com/example/demo/repository/ReservationRepository.java`

**目的:** 予約編集時の重複チェック用クエリを追加（自分自身の予約は除外）

```diff
 	@Query("SELECT COUNT(r) FROM Reservation r WHERE r.user.id = :userId AND r.plan.accommodation.id = :acmId AND r.isCancelled = false AND r.checkInDate < :checkOut AND r.checkOutDate > :checkIn")
 	long countDuplicateReservations(
 			@Param("userId") Integer userId,
 			@Param("acmId") Integer acmId,
 			@Param("checkIn") LocalDate checkIn,
 			@Param("checkOut") LocalDate checkOut);

+	// 予約編集時の重複チェック（自分自身の予約ID(resId)を除外、すべての宿を対象）
+	@Query("SELECT COUNT(r) FROM Reservation r WHERE r.user.id = :userId AND r.id <> :resId AND r.isCancelled = false AND r.checkInDate < :checkOut AND r.checkOutDate > :checkIn")
+	long countDuplicateReservationsForEdit(
+			@Param("userId") Integer userId,
+			@Param("resId") Integer resId,
+			@Param("checkIn") LocalDate checkIn,
+			@Param("checkOut") LocalDate checkOut);

 	Boolean existsByUserIdAndCheckInDateGreaterThanEqual(Integer userId, LocalDate checkInDate);
```

---

## 2. ReservationService.java

**対象ファイル:** `src/main/java/com/example/demo/service/ReservationService.java`

**目的:** `editReservation` の戻り値を `boolean` → `int` に変更し、重複チェックを追加

### 変更① メソッドシグネチャの変更（114行目付近）

```diff
-	public boolean editReservation(Integer userId, Integer planId, Integer resId, LocalDate checkIn, LocalDate checkOut,
+	public int editReservation(Integer userId, Integer planId, Integer resId, LocalDate checkIn, LocalDate checkOut,
 			Integer peopleCount, Integer roomCount) {
```

### 変更② 在庫不足時の戻り値を変更（138〜139行目付近）

```diff
 		if (newstocks.size() != stayDays) {
 			rollbackOldStocks(oldstocks, res.getRoomCount());
-			return false;
+			return 2;
 		}
```

### 変更③ 在庫不足（部屋数）時の戻り値を変更（143〜147行目付近）

```diff
 		for (PlanStock newstock : newstocks) {
 			if (newstock.getStock() < roomCount) {
 				rollbackOldStocks(oldstocks, res.getRoomCount());
-				return false;
+				return 2;
 			}
 		}
```

### 変更④ 重複予約チェックを追加（コメント行「// 宿泊期間が現在の予約とかぶっていないかチェック」の直後）

**変更前（150〜151行目）:**
```java
		// 宿泊期間が現在の予約とかぶっていないかチェック

		// 在庫の減算
```

**変更後:**
```java
		// 宿泊期間が現在の予約とかぶっていないかチェック（自分以外の全予約を対象）
		long dupCount = reservationRepository.countDuplicateReservationsForEdit(userId, resId, checkIn, checkOut);
		if (dupCount > 0) {
			rollbackOldStocks(oldstocks, res.getRoomCount());
			return 3;
		}

		// 在庫の減算
```

### 変更⑤ 成功時の戻り値を変更（170〜171行目付近）

```diff
 		reservationRepository.save(res);
-		return true;
+		return 1;
 	}
```

### 変更⑥ importの追加（ファイル先頭のimport部分）

`countDuplicateReservationsForEdit` は既存の `reservationRepository` を使うので追加のimportは不要です。

---

## 3. AdminResController.java

**対象ファイル:** `src/main/java/com/example/demo/controller/AdminResController.java`

**目的:** `update` メソッドにサーバー側バリデーションを追加し、重複予約エラーメッセージを出す

**変更前（update メソッド全体、98〜123行目）:**
```java
	// 予約の編集処理
	@PostMapping("/admin/reservations/edit/{resId}")
	public String update(
			@PathVariable Integer resId,
			@RequestParam(defaultValue = "") Integer planId,
			@RequestParam(defaultValue = "") LocalTime checkInTime,
			@RequestParam(defaultValue = "") LocalTime checkOutTime,
			@RequestParam(required = false) LocalDate checkInDate,
			@RequestParam(required = false) LocalDate checkOutDate,
			@RequestParam(defaultValue = "") Integer peopleCount,
			@RequestParam(defaultValue = "") Integer roomCount,
			RedirectAttributes redirectAttributes,
			Model model) {
		Reservation targetRes = reservationRepository.findById(resId).get();
		Integer userId = targetRes.getUser().getId();
		planId = targetRes.getPlan().getId();
		boolean success = reservationService.editReservation(userId, planId, resId, checkInDate, checkOutDate,
				peopleCount, roomCount);

		if (!(success)) {
			redirectAttributes.addFlashAttribute("errorMsg", "指定された日程で部屋が確保できませんでした。");
			model.addAttribute("res", targetRes);
			return "redirect:/admin/reservations/edit/" + resId;
		}

		return "redirect:/admin/reservations/detail/" + resId;
	}
```

**変更後:**
```java
	// 予約の編集処理
	@PostMapping("/admin/reservations/edit/{resId}")
	public String update(
			@PathVariable Integer resId,
			@RequestParam(defaultValue = "") Integer planId,
			@RequestParam(defaultValue = "") LocalTime checkInTime,
			@RequestParam(defaultValue = "") LocalTime checkOutTime,
			@RequestParam(required = false) LocalDate checkInDate,
			@RequestParam(required = false) LocalDate checkOutDate,
			@RequestParam(defaultValue = "") Integer peopleCount,
			@RequestParam(defaultValue = "") Integer roomCount,
			RedirectAttributes redirectAttributes,
			Model model) {

		// サーバー側バリデーション
		Reservation targetRes = reservationRepository.findById(resId).get();

		// 空欄チェック
		if (checkInDate == null || checkOutDate == null || peopleCount == null || roomCount == null) {
			redirectAttributes.addFlashAttribute("errorMsg", "空欄の項目があります");
			return "redirect:/admin/reservations/edit/" + resId;
		}

		// 過去日チェック
		if (checkInDate.isBefore(LocalDate.now())) {
			redirectAttributes.addFlashAttribute("errorMsg", "チェックイン日は明日以降を指定してください");
			return "redirect:/admin/reservations/edit/" + resId;
		}

		// チェックアウト日がチェックイン日より後かチェック
		if (!checkOutDate.isAfter(checkInDate)) {
			redirectAttributes.addFlashAttribute("errorMsg", "チェックアウト日はチェックイン日より後を指定してください");
			return "redirect:/admin/reservations/edit/" + resId;
		}

		// 宿泊人数の上限・下限チェック
		if (peopleCount < 1) {
			redirectAttributes.addFlashAttribute("errorMsg", "宿泊人数は1人以上を指定してください");
			return "redirect:/admin/reservations/edit/" + resId;
		}
		if (peopleCount > targetRes.getPlan().getPeopleCount()) {
			redirectAttributes.addFlashAttribute("errorMsg", "宿泊人数が定員人数を超えています");
			return "redirect:/admin/reservations/edit/" + resId;
		}

		Integer userId = targetRes.getUser().getId();
		planId = targetRes.getPlan().getId();
		int result = reservationService.editReservation(userId, planId, resId, checkInDate, checkOutDate,
				peopleCount, roomCount);

		if (result == 2) {
			redirectAttributes.addFlashAttribute("errorMsg", "指定された日程で部屋が確保できませんでした。");
			return "redirect:/admin/reservations/edit/" + resId;
		}
		if (result == 3) {
			redirectAttributes.addFlashAttribute("errorMsg", "選択された日程にはすでに予約が登録されています");
			return "redirect:/admin/reservations/edit/" + resId;
		}

		return "redirect:/admin/reservations/detail/" + resId;
	}
```

> [!NOTE]
> `LocalDate` のimportはすでに存在するので追加不要です。

---

## 4. UserController.java

**対象ファイル:** `src/main/java/com/example/demo/controller/UserController.java`

**目的:** 退会処理でセッションを破棄する（`session.invalidate()`）

### 変更① メソッドシグネチャに `HttpSession session` を追加（162〜165行目）

```diff
-	@PostMapping("/users/delete/{userId}")
-	public String delete(
-			@PathVariable Integer userId,
-			RedirectAttributes redirectAttributes) {
+	@PostMapping("/users/delete/{userId}")
+	public String delete(
+			@PathVariable Integer userId,
+			RedirectAttributes redirectAttributes,
+			HttpSession session) {
```

### 変更② else ブロックでセッションを破棄（171〜174行目）

```diff
 		} else {
 			User user = userRepository.findById(userId).get();
 			user.setDeactivatedAt(LocalDate.now());
 			userRepository.save(user);
+			session.invalidate();
 			return "user/login";
 		}
```

---

## 5. MyPageController.java

**対象ファイル:** `src/main/java/com/example/demo/controller/MyPageController.java`

### 変更① `updateProfile` — 成功時のリダイレクト修正 + メールアドレス重複チェック追加

**変更前（74〜116行目）:**
```java
		// エラーチェック
		List<String> errorList = new ArrayList<>();
		if (familyName.length() == 0) {
			errorList.add("名前(姓)は必須です");
		}
		if (givenName.length() == 0) {
			errorList.add("名前(名)は必須です");
		}
		if (email.length() == 0) {
			errorList.add("メールアドレスは必須です");
		}
		// ... (途中省略)

		if (errorList.size() > 0) {
			redirectAttributes.addFlashAttribute("errorList", errorList);
			return "redirect:/mypage/edit/profile/" + userId;
		}

		User user = userRepository.findById(userId).get();
		// ... (フィールド更新省略)

		redirectAttributes.addFlashAttribute("msg", "登録完了しました");
		userRepository.save(user);

		return "accommodation/search";  // ← ここが問題
```

**変更後（差分のみ）:**

エラーチェックブロックに以下を追加（`email.length() == 0` チェックの直後）：
```java
		// メールアドレス重複チェック（自分以外に同じメールアドレスが存在するか）
		List<User> existingUsers = userRepository.findByEmail(email);
		if (existingUsers != null && existingUsers.stream().anyMatch(u -> !u.getId().equals(userId))) {
			errorList.add("登録済みのメールアドレスです");
		}
```

最後の `return "accommodation/search";` を以下に変更：
```diff
-		return "accommodation/search";
+		return "redirect:/accommodations";
```

### 変更② `updateUser` — 成功時のリダイレクト修正

**変更前（152〜154行目）:**
```java
		User user = userRepository.findById(account.getId()).get();
		user.setPassword(password);
		userRepository.save(user);
		return "accommodation/search";  // ← ここが問題
```

**変更後:**
```diff
 		User user = userRepository.findById(account.getId()).get();
 		user.setPassword(password);
 		userRepository.save(user);
-		return "accommodation/search";
+		return "redirect:/accommodations";
```

---

## 6. AdminController.java

**対象ファイル:** `src/main/java/com/example/demo/controller/AdminController.java`

**目的:** ログアウトURLのタイポ修正（`"admin/logout"` → `"/admin/logout"`）

**変更箇所（36行目）:**
```diff
-	@GetMapping({ "/admin/login", "admin/logout" })
+	@GetMapping({ "/admin/login", "/admin/logout" })
 	public String index() {
```

---

## 7. AccommodationController.java

**対象ファイル:** `src/main/java/com/example/demo/controller/AccommodationController.java`

**目的:** 日付未入力の場合でも、都道府県名・宿名での絞り込み検索を機能させる

**変更箇所（`show` メソッド内、94〜107行目）:**

```diff
 		List<AccommodationDto> acmList;
 		if (acmSearch.getCheckInDate() != null && acmSearch.getCheckOutDate() != null) {
 			innDays = (int) ChronoUnit.DAYS.between(acmSearch.getCheckInDate(), acmSearch.getCheckOutDate());
 			acmList = acmRepository.findByAllSearch(
 					acmSearch.getKeyword(),
 					acmSearch.getPrefectureId(),
 					acmSearch.getCategoryId(),
 					acmSearch.getCheckInDate(),
 					acmSearch.getCheckOutDate(),
 					acmSearch.getPeopleCount(),
 					innDays);
 		} else {
-			acmList = acmRepository.findAllDto();
+			// 日付なしでもキーワード・都道府県・カテゴリーで絞り込む
+			acmList = acmRepository.findByAllSearch(
+					acmSearch.getKeyword(),
+					acmSearch.getPrefectureId(),
+					acmSearch.getCategoryId(),
+					null,
+					null,
+					acmSearch.getPeopleCount(),
+					0);
 		}
```

> [!IMPORTANT]
> この修正が動作するためには、`AcmRepository.java` の `findByAllSearch` クエリで `checkIn` と `checkOut` が `null` のとき在庫チェックをスキップする必要があります。
>
> **AcmRepository.java の JPQL クエリの在庫件数サブクエリ部分を変更：**
>
> ```diff
> -+ "AND ps2.targetDate BETWEEN :checkIn AND :checkOut "
> -+ ") > :innDays "
> ++ "AND (cast(:checkIn as localdate) IS NULL OR ps2.targetDate BETWEEN :checkIn AND :checkOut) "
> ++ ") > :innDays "
> ```
>
> ただし、HibernateのJPQLでは `BETWEEN` に `null` を渡すと挙動が不定になるため、代替手段として **日付が空の場合に在庫チェックをバイパスする** ように条件式を変更します。具体的には：
>
> ```diff
> -"(SELECT COUNT(*) "
> -+ "FROM PlanStock ps2 "
> -+ "WHERE "
> -+ "ps1.plan.id = ps2.plan.id "
> -+ "AND COALESCE(ps2.stock , 0) > 0 "
> -+ "AND ps2.targetDate BETWEEN :checkIn AND :checkOut "
> -+ ") > :innDays "
> +"(cast(:checkIn as localdate) IS NULL OR "
> +"(SELECT COUNT(*) "
> +"FROM PlanStock ps2 "
> +"WHERE "
> +"ps1.plan.id = ps2.plan.id "
> +"AND COALESCE(ps2.stock , 0) > 0 "
> +"AND ps2.targetDate BETWEEN :checkIn AND :checkOut "
> +") > :innDays) "
> ```

---

## 修正ファイル一覧まとめ

| # | ファイル | 修正内容 | 優先度 |
|:--|:---|:---|:---:|
| 1 | `ReservationRepository.java` | `countDuplicateReservationsForEdit` クエリ追加 | ★★★ |
| 2 | `ReservationService.java` | `editReservation` 戻り値を `int` に変更 + 重複チェック追加 | ★★★ |
| 3 | `AdminResController.java` | バリデーション追加 + int 戻り値対応 | ★★★ |
| 4 | `UserController.java` | 退会時に `session.invalidate()` 追加 | ★★ |
| 5 | `MyPageController.java` | リダイレクト修正 + メール重複チェック追加 | ★★ |
| 6 | `AdminController.java` | ログアウトURLの `/` タイポ修正 | ★ |
| 7 | `AccommodationController.java` + `AcmRepository.java` | 日付なし検索対応 | ★★ |
