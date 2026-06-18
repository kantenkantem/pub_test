# 修正内容ウォークスルー（手動適用用・改善版）

> [!IMPORTANT]
> すべてのファイルは **手動で** 変更してください。
> 以前のウォークスルーにあったJPQLクエリのキャストエラーの危険性や、結合テスト仕様書とのメッセージ乖離、および退会処理時のバグを修正した「改善版」です。

---

## 1. ReservationRepository.java

**対象ファイル:** `src/main/java/com/example/demo/repository/ReservationRepository.java`

### 変更① 予約編集時の重複チェック用クエリを追加（33〜38行目付近の後に追加）

自分以外の有効な予約で期間が重複するものをカウントします。

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
+
 	Boolean existsByUserIdAndCheckInDateGreaterThanEqual(Integer userId, LocalDate checkInDate);
```

### 変更② 退会チェック用クエリの追加（最終行）

キャンセル済みの予約を退会制限から除外するため、`AndIsCancelledFalse` を追加したメソッドを追加します。

```diff
-	Boolean existsByUserIdAndCheckInDateGreaterThanEqual(Integer userId, LocalDate checkInDate);
+	Boolean existsByUserIdAndCheckInDateGreaterThanEqual(Integer userId, LocalDate checkInDate);
+	Boolean existsByUserIdAndCheckInDateGreaterThanEqualAndIsCancelledFalse(Integer userId, LocalDate checkInDate);
 }
```

---

## 2. ReservationService.java

**対象ファイル:** `src/main/java/com/example/demo/service/ReservationService.java`

### 変更① メソッドシグネチャの変更（114行目付近）

戻り値を `boolean` → `int` に変更します。

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

---

## 3. AdminResController.java

**対象ファイル:** `src/main/java/com/example/demo/controller/AdminResController.java`

### 変更① `update` メソッドのバリデーションとメッセージを結合テスト仕様書に完全一致させる

**変更前（98〜123行目）:**
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

		Reservation targetRes = reservationRepository.findById(resId).get();

		// 結合テスト仕様書(UC705-3)に基づく必須項目（空欄）バリデーション
		if (checkInDate == null) {
			redirectAttributes.addFlashAttribute("errorMsg", "チェックイン日は必須です");
			return "redirect:/admin/reservations/edit/" + resId;
		}
		if (checkOutDate == null) {
			redirectAttributes.addFlashAttribute("errorMsg", "チェックアウト日は必須です");
			return "redirect:/admin/reservations/edit/" + resId;
		}
		if (peopleCount == null || peopleCount < 1) {
			redirectAttributes.addFlashAttribute("errorMsg", "宿泊人数は1人以上で入力してください");
			return "redirect:/admin/reservations/edit/" + resId;
		}
		if (roomCount == null) {
			redirectAttributes.addFlashAttribute("errorMsg", "空欄の項目があります");
			return "redirect:/admin/reservations/edit/" + resId;
		}

		// 過去日チェック
		if (checkInDate.isBefore(LocalDate.now())) {
			redirectAttributes.addFlashAttribute("errorMsg", "チェックイン日は明日以降を指定してください");
			return "redirect:/admin/reservations/edit/" + resId;
		}

		// チェックアウト日整合性チェック
		if (!checkOutDate.isAfter(checkInDate)) {
			redirectAttributes.addFlashAttribute("errorMsg", "チェックアウト日はチェックイン日より後を指定してください");
			return "redirect:/admin/reservations/edit/" + resId;
		}

		// 定員上限チェック (UC705-8)
		if (peopleCount > targetRes.getPlan().getPeopleCount()) {
			redirectAttributes.addFlashAttribute("errorMsg", "宿泊人数はプランの定員人数内で入力してください");
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

---

## 4. UserController.java

**対象ファイル:** `src/main/java/com/example/demo/controller/UserController.java`

### 変更① 退会時のセッション破棄・リダイレクト修正およびキャンセル済みの予約を考慮したチェック

**変更前（161〜176行目）:**
```java
	// 退会処理
	@PostMapping("/users/delete/{userId}")
	public String delete(
			@PathVariable Integer userId,
			RedirectAttributes redirectAttributes) {
		if (reservationRepository.existsByUserIdAndCheckInDateGreaterThanEqual(userId,
				LocalDate.now())) {
			redirectAttributes.addFlashAttribute("notDeleteMsg", "キャンセルされていない予約があるため、退会できません");
			return "redirect:/mypage/edit/profile/" + userId;
		} else {
			User user = userRepository.findById(userId).get();
			user.setDeactivatedAt(LocalDate.now());
			userRepository.save(user);
			return "user/login";
		}
	}
```

**変更後:**
```java
	// 退会処理
	@PostMapping("/users/delete/{userId}")
	public String delete(
			@PathVariable Integer userId,
			RedirectAttributes redirectAttributes,
			HttpSession session) {
		// キャンセルされていない有効な予約のみを対象にチェック (exists...AndIsCancelledFalseを使用)
		if (reservationRepository.existsByUserIdAndCheckInDateGreaterThanEqualAndIsCancelledFalse(userId,
				LocalDate.now())) {
			redirectAttributes.addFlashAttribute("notDeleteMsg", "キャンセルされていない予約があるため、退会できません");
			return "redirect:/mypage/edit/profile/" + userId;
		} else {
			User user = userRepository.findById(userId).get();
			user.setDeactivatedAt(LocalDate.now());
			userRepository.save(user);
			
			// セッションを無効化し、ログイン画面へリダイレクト
			session.invalidate();
			return "redirect:/login";
		}
	}
```

---

## 5. MyPageController.java

**対象ファイル:** `src/main/java/com/example/demo/controller/MyPageController.java`

### 変更① `updateProfile` — 成功時のリダイレクト修正 + メールアドレス重複チェック追加

**変更箇所（74〜116行目付近）:**

エラーチェックブロックに以下を追加（`email.length() == 0` チェックの直後）：
```java
		// メールアドレス重複チェック（自分以外に同じメールアドレスが存在するか）
		List<User> existingUsers = userRepository.findByEmail(email);
		if (existingUsers != null && existingUsers.stream().anyMatch(u -> !u.getId().equals(userId))) {
			errorList.add("登録済みのメールアドレスです");
		}
```

最後の戻り値を論理名からリダイレクトに変更：
```diff
-		return "accommodation/search";
+		return "redirect:/accommodations";
```

### 変更② `updateUser` — 成功時のリダイレクト修正

**変更箇所（154行目付近）:**

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

## 7. AccommodationController.java & AcmRepository.java

JPQLの中で `cast` や `null` チェックを記述すると、インメモリデータベース（H2など）やSQLのパース時に構文エラーになる危険性が高いため、**日付を含まない（在庫チェックをしない）検索メソッドを別で追加**します。

### 変更① AcmRepository.java に `findByAllSearchWithoutDates` を追加

**対象ファイル:** `src/main/java/com/example/demo/repository/AcmRepository.java`

`findByAllSearch` メソッドの直下などに追加します。

```java
	@Query("SELECT new com.example.demo.dto.AccommodationDto"
			+ "("
			+ "ps1.plan.accommodation.id,"
			+ "ps1.plan.accommodation.name,"
			+ "ps1.plan.accommodation.acmCategory.name,"
			+ "CONCAT(ps1.plan.accommodation.prefecture.name , ps1.plan.accommodation.address),"
			+ "MIN(ps1.plan.taxIncludePrice)"
			+ ") "
			+ "FROM PlanStock ps1 "
			+ "WHERE "
			+ "(:prefId IS NULL OR ps1.plan.accommodation.prefecture.id = :prefId) "
			+ "AND (:categoryId IS NULL OR ps1.plan.accommodation.acmCategory.id = :categoryId) "
			+ "AND (:name IS NULL OR :name = '' OR ps1.plan.accommodation.name LIKE CONCAT('%', :name , '%')) "
			+ "AND (:peopleCount IS NULL OR ps1.plan.peopleCount >= :peopleCount) "
			+ "GROUP BY "
			+ "ps1.plan.accommodation.id,"
			+ "ps1.plan.accommodation.name,"
			+ "ps1.plan.accommodation.acmCategory.name,"
			+ "CONCAT(ps1.plan.accommodation.prefecture.name , ps1.plan.accommodation.address)")
	List<AccommodationDto> findByAllSearchWithoutDates(
			@Param("name") String name,
			@Param("prefId") Integer prefectureId,
			@Param("categoryId") Integer categoryId,
			@Param("peopleCount") Integer peopleCount);
```

### 変更② AccommodationController.java の show メソッドを書き換え

**対象ファイル:** `src/main/java/com/example/demo/controller/AccommodationController.java`

**変更前（94〜107行目）:**
```java
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
			acmList = acmRepository.findAllDto();
		}
```

**変更後:**
```java
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
			// 日付指定がない場合は、日付条件および在庫条件を除いた絞り込みメソッドを呼び出す
			acmList = acmRepository.findByAllSearchWithoutDates(
					acmSearch.getKeyword(),
					acmSearch.getPrefectureId(),
					acmSearch.getCategoryId(),
					acmSearch.getPeopleCount());
		}
```
