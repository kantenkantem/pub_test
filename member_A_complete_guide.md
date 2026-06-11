# メンバーA (テクニカルリーダー) 専用：宿予約システム完全実装マニュアル (最終日同期版)

本ドキュメントは、6月11日に他のメンバーがコミットした最新のファイル構成（`AdminAcmController`等）、メソッド名、および引数の型に完全同期させた「テクニカルリーダー専用の完全コード集」です。

また、**あなた（メンバーA）が実装した `PlanService` および `ReservationService` を、他のメンバーがコントローラーでどのように呼び出して使うべきかの具体的なコード指示**を第3章に追加しています。他メンバーのサポートや完成時のマージ調整に活用してください。

---

## 目次
1. [リポジトリ層の定義 (追加すべき全メソッド一覧)](#1-リポジトリ層の定義-追加すべき全メソッド一覧)
2. [サービス層の定義 (メンバーA実装：PlanService / ReservationService)](#2-サービス層の定義-メンバーa実装planservice--reservationservice)
3. [【重要】メンバーA作成サービスを他メンバーが呼び出す方法 (使用指示)](#3-重要メンバーa作成サービスを他メンバーが呼び出す方法-使用指示)
4. [会員側コントローラー完全同期コード (UserController)](#4-会員側コントローラー完全同期コード-usercontroller)
5. [会員側検索・予約コントローラー完全同期コード (AccommodationController / ReservationController)](#5-会員側検索予約コントローラー完全同期コード-accommodationcontroller--reservationcontroller)
6. [管理者側コントローラー完全同期コード (AdminAcmController / AdminPlanController / AdminResController / AdminUserController / AdminController)](#6-管理者側コントローラー完全同期コード)
7. [Thymeleaf HTML連携・バインド対応表](#7-thymeleaf-html連携バインド対応表)

---

## 1. リポジトリ層の定義 (追加すべき全メソッド一覧)
6月11日時点の最新の Entity 構造（`User` に `Prefecture` が入るなど）に合わせたクエリメソッドの定義です。

#### `UserRepository.java`
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.User;
import java.util.List;

public interface UserRepository extends JpaRepository<User, Integer> {
    List<User> findByEmail(String email);
    List<User> findByEmailAndPassword(String email, String password);
    List<User> findByFamilyNameContainingOrGivenNameContaining(String familyName, String givenName);
}
```

#### `AcmRepository.java`
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.Accommodation;
import java.util.List;

public interface AcmRepository extends JpaRepository<Accommodation, Integer> {
    List<Accommodation> findByPrefectureId(Integer prefectureId);
    List<Accommodation> findByNameContaining(String name);
    List<Accommodation> findByPrefectureIdAndNameContaining(Integer prefectureId, String name);
}
```

#### `PlanRepository.java`
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.Plan;
import java.util.List;

public interface PlanRepository extends JpaRepository<Plan, Integer> {
    List<Plan> findByAccommodationId(Integer accommodationId);
}
```

#### `PlanStockRepository.java`
```java
package com.example.demo.repository;

import java.time.LocalDate;
import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.PlanStock;

public interface PlanStockRepository extends JpaRepository<PlanStock, Integer> {
    // 修正: Entityの「plan」リレーションに対応し、JPAの命名規則で自動結合させます
    List<PlanStock> findByPlanIdAndTargetDateBetween(Integer planId, LocalDate start, LocalDate end);
}
```

#### `ReservationRepository.java`
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.Reservation;
import java.util.List;

public interface ReservationRepository extends JpaRepository<Reservation, Integer> {
    List<Reservation> findByUserIdOrderByReservedAtDesc(Integer userId);
}
```

---

## 2. サービス層の定義 (メンバーA実装：PlanService / ReservationService)
トランザクション処理と在庫管理をカプセル化した、あなたが担当するクラスです。

#### `PlanService.java`
```java
package com.example.demo.service;

import java.time.LocalDate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.example.demo.entity.Plan;
import com.example.demo.entity.PlanStock;
import com.example.demo.repository.PlanRepository;
import com.example.demo.repository.PlanStockRepository;

@Service
public class PlanService {

    private final PlanRepository planRepository;
    private final PlanStockRepository planStockRepository;

    public PlanService(PlanRepository planRepository, PlanStockRepository planStockRepository) {
        this.planRepository = planRepository;
        this.planStockRepository = planStockRepository;
    }

    @Transactional
    public void createNewPlan(Plan plan, Integer initialStock) {
        Plan savedPlan = planRepository.save(plan);

        // 今日を起点に90日分の在庫レコードを生成
        LocalDate today = LocalDate.now();
        for (int i = 0; i < 90; i++) {
            PlanStock stock = new PlanStock();
            stock.setPlan(savedPlan); // リレーションオブジェクトを設定
            stock.setTargetDate(today.plusDays(i));
            stock.setStock(initialStock);
            planStockRepository.save(stock);
        }
    }
}
```

#### `ReservationService.java`
```java
package com.example.demo.service;

import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.List;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.example.demo.entity.User;
import com.example.demo.entity.Plan;
import com.example.demo.entity.Reservation;
import com.example.demo.entity.PlanStock;
import com.example.demo.repository.UserRepository;
import com.example.demo.repository.PlanRepository;
import com.example.demo.repository.ReservationRepository;
import com.example.demo.repository.PlanStockRepository;

@Service
public class ReservationService {

    private final UserRepository userRepository;
    private final PlanRepository planRepository;
    private final ReservationRepository reservationRepository;
    private final PlanStockRepository planStockRepository;

    public ReservationService(UserRepository userRepository, PlanRepository planRepository,
                              ReservationRepository reservationRepository, PlanStockRepository planStockRepository) {
        this.userRepository = userRepository;
        this.planRepository = planRepository;
        this.reservationRepository = reservationRepository;
        this.planStockRepository = planStockRepository;
    }

    @Transactional
    public boolean processReservation(Integer userId, Integer planId, LocalDate checkIn, LocalDate checkOut,
                                      Integer peopleCount, Integer roomCount) {
        
        LocalDate lastNight = checkOut.minusDays(1);
        List<PlanStock> stocks = planStockRepository.findByPlanIdAndTargetDateBetween(planId, checkIn, lastNight);

        long stayDays = java.time.temporal.ChronoUnit.DAYS.between(checkIn, checkOut);
        if (stocks.size() != stayDays) {
            return false; // 在庫情報が不足
        }

        for (PlanStock stock : stocks) {
            if (stock.getStock() < roomCount) {
                return false; // 満室
            }
        }

        // 在庫の減算
        for (PlanStock stock : stocks) {
            stock.setStock(stock.getStock() - roomCount);
            planStockRepository.save(stock);
        }

        // 予約の登録
        User user = userRepository.findById(userId).orElseThrow();
        Plan plan = planRepository.findById(planId).orElseThrow();

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
        return true;
    }

    @Transactional
    public void cancelReservationByAdmin(Integer reservationId) {
        Reservation res = reservationRepository.findById(reservationId).orElseThrow();

        if (res.getIsCancelled()) return;

        res.setIsCancelled(true);
        reservationRepository.save(res);

        // 在庫の復元
        LocalDate lastNight = res.getCheckOutDate().minusDays(1);
        List<PlanStock> stocks = planStockRepository.findByPlanIdAndTargetDateBetween(
                res.getPlan().getId(), res.getCheckInDate(), lastNight
        );

        for (PlanStock stock : stocks) {
            stock.setStock(stock.getStock() + res.getRoomCount());
            planStockRepository.save(stock);
        }
    }
}
```

---

## 3. 【重要】メンバーA作成サービスを他メンバーが呼び出す方法 (使用指示)

あなたが作成した `PlanService` と `ReservationService` は、他のメンバーが各コントローラーにインジェクションして呼び出すだけで、登録や在庫制御が自動的に実行されます。各コントローラーでの具体的な使い方と指示です。

### 3.1 メンバーDへの指示（プラン登録時に `PlanService` を使う）
* **呼び出し先**: [AdminPlanController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AdminPlanController.java) の `store` メソッド
* **使い方**:
  1. コントローラーのコンストラクタ引数に `PlanService` を追加し、フィールドに保持します。
  2. プラン追加POST（`store`）メソッドの中で `planService.createNewPlan(plan, roomCount)` を呼び出します。これにより、DB保存と同時に90日分の在庫が自動インサートされます。
```java
// コントローラー内のインジェクション指示
private final PlanService planService;

public AdminPlanController(AcmRepository acmRepository, PlanRepository planRepository, PlanService planService) {
    this.acmRepository = acmRepository;
    this.planRepository = planRepository;
    this.planService = planService; // DIによる受け取り
}

@PostMapping("/plans/add/{acmId}")
public String store(
        @PathVariable Integer acmId,
        @RequestParam String name,
        @RequestParam String content,
        @RequestParam Integer taxExcludePrice,
        @RequestParam Integer roomCount,
        @RequestParam Integer peopleCount,
        @RequestParam Boolean isBreakfast) {

    Accommodation accommodation = acmRepository.findById(acmId).get();
    
    // 税込価格の計算とオブジェクト生成（Entity定義に合わせる）
    int taxIncludePrice = (int) (taxExcludePrice * 1.1);
    Plan plan = new Plan(accommodation, name, content, taxExcludePrice, taxIncludePrice, roomCount, peopleCount, false, isBreakfast);

    // 呼び出し指示: メンバーA作成のサービスで保存と在庫生成を行う
    planService.createNewPlan(plan, roomCount);

    return "redirect:/admin/accommodations/detail/" + acmId;
}
```

### 3.2 メンバーCへの指示（一般予約確定時に `ReservationService` を使う）
* **呼び出し先**: [ReservationController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/ReservationController.java) の `register` メソッド
* **注意点**: メンバーCの `register` アクションメソッドのパラメータには、**`roomCount` (部屋数) が抜け落ちています**。必ず `@RequestParam` に追加させ、サービスを呼び出すように指示してください。
* **使い方**:
```java
// コントローラーへのインジェクション指示
private final ReservationService reservationService;

public ReservationController(ReservationRepository reservationRepository, PlanRepository planRepository, ReservationService reservationService) {
    this.reservationRepository = reservationRepository;
    this.planRepository = planRepository;
    this.reservationService = reservationService;
}

@PostMapping("/reservations/confirm")
public String register(
        @RequestParam Integer planId,
        @RequestParam Integer userId,
        @RequestParam LocalDate checkIndate, // 他メンバーが定義したLocalDate型に合わせる
        @RequestParam LocalDate checkOutDate,
        @RequestParam Integer peopleCount,
        @RequestParam(defaultValue = "1") Integer roomCount, // 修正: 追加指示
        Model model) {

    // 呼び出し指示: 在庫チェック・減算・予約登録をトランザクション内で実行
    boolean success = reservationService.processReservation(
            userId, planId, checkIndate, checkOutDate, peopleCount, roomCount
    );

    if (!success) {
        model.addAttribute("message", "空室が足りないため、予約を確定できませんでした。");
        return "reserve/confirm"; // エラーメッセージ付きで確認画面に戻す
    }

    return "redirect:/mypage"; // 成功時は予約履歴一覧（マイページ）へ
}
```

### 3.3 メンバーEへの指示（管理者代理キャンセル時に `ReservationService` を使う）
* **呼び出し先**: [AdminResController.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/controller/AdminResController.java) の `archiveRes` メソッド
* **使い方**:
```java
// コントローラーへのインジェクション指示
private final ReservationService reservationService;

public AdminResController(ReservationRepository reservationRepository, UserRepository userRepository, ReservationService reservationService) {
    this.reservationRepository = reservationRepository;
    this.userRepository = userRepository;
    this.reservationService = reservationService;
}

@PostMapping("/reservations/archive/{resId}")
public String archiveRes(@PathVariable int resId) {
    // 呼び出し指示: キャンセルフラグ更新と在庫の復元を同時に行う
    reservationService.cancelReservationByAdmin(resId);
    
    // リダイレクト先を正しいURLに修正
    return "redirect:/admin/reservations"; 
}
```

---

## 4. 会員側コントローラー完全同期コード (UserController)
`MypageController` の内容を完全に統合した `UserController` です。

#### `UserController.java` (メンバーB担当範囲)
```java
package com.example.demo.controller;

import java.time.LocalDate;
import java.util.List;
import java.util.Optional;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.User;
import com.example.demo.entity.Reservation;
import com.example.demo.entity.Prefecture;
import com.example.demo.repository.UserRepository;
import com.example.demo.repository.ReservationRepository;
import com.example.demo.repository.PrefectureRepository;

@Controller
public class UserController {

    private final UserRepository userRepository;
    private final ReservationRepository reservationRepository;
    private final PrefectureRepository prefectureRepository;

    public UserController(UserRepository userRepository, ReservationRepository reservationRepository, PrefectureRepository prefectureRepository) {
        this.userRepository = userRepository;
        this.reservationRepository = reservationRepository;
        this.prefectureRepository = prefectureRepository;
    }

    @GetMapping("/login")
    public String showLoginForm() {
        return "user/login";
    }

    @PostMapping("/login")
    public String login(
            @RequestParam("email") String email,
            @RequestParam("password") String password,
            HttpSession session,
            Model model) {
        
        Optional<User> userOpt = userRepository.findByEmail(email).stream().findFirst();
        
        if (userOpt.isPresent()) {
            User user = userOpt.get();
            if (!user.getPassword().equals(password)) {
                model.addAttribute("message", "メールアドレス、パスワードが一致しませんでした");
                return "user/login";
            }
            if (user.getDeactivatedAt() != null) {
                model.addAttribute("message", "このアカウントは退会済みです");
                return "user/login";
            }
            session.setAttribute("loginUser", user);
            return "redirect:/mypage";
        } else {
            model.addAttribute("message", "メールアドレス、パスワードが一致しませんでした");
            return "user/login";
        }
    }

    @GetMapping("/logout")
    public String logout(HttpSession session) {
        session.removeAttribute("loginUser");
        return "redirect:/login";
    }

    @GetMapping("/users/add")
    public String showRegisterForm(Model model) {
        model.addAttribute("pref", prefectureRepository.findAll());
        return "user/register";
    }

    @PostMapping("/users/add")
    public String register(
            @RequestParam("familyName") String familyName,
            @RequestParam("givenName") String givenName,
            @RequestParam("email") String email,
            @RequestParam("postalCode") String postalCode,
            @RequestParam("prefectureId") Integer prefectureId,
            @RequestParam("address") String address,
            @RequestParam("phoneNumber") String phoneNumber,
            @RequestParam("birthDate") String birthDateStr,
            @RequestParam("password") String password,
            Model model) {

        if (!userRepository.findByEmail(email).isEmpty()) {
            model.addAttribute("message", "登録済みのメールアドレスです");
            model.addAttribute("pref", prefectureRepository.findAll());
            return "user/register";
        }

        Prefecture pref = prefectureRepository.findById(prefectureId).orElseThrow();

        User user = new User(familyName, givenName, email, password, postalCode, pref, address, phoneNumber, LocalDate.parse(birthDateStr));
        user.setRegisteredAt(LocalDate.now());

        userRepository.save(user);
        return "redirect:/login";
    }

    @GetMapping("/mypage")
    public String showMypage(HttpSession session, Model model) {
        User loginUser = (User) session.getAttribute("loginUser");
        if (loginUser == null) {
            return "redirect:/login";
        }

        User user = userRepository.findById(loginUser.getId()).orElse(loginUser);
        List<Reservation> reservations = reservationRepository.findByUserIdOrderByReservedAtDesc(user.getId());
        
        model.addAttribute("user", user);
        model.addAttribute("reservations", reservations);
        return "user/mypage";
    }

    @PostMapping("/users/deactivate")
    public String deactivateUser(HttpSession session) {
        User loginUser = (User) session.getAttribute("loginUser");
        if (loginUser == null) {
            return "redirect:/login";
        }

        User user = userRepository.findById(loginUser.getId()).orElseThrow();
        user.setDeactivatedAt(LocalDate.now());
        userRepository.save(user);

        session.removeAttribute("loginUser");
        return "redirect:/login";
    }
}
```

---

## 5. 会員側検索・予約コントローラー完全同期コード

#### `AccommodationController.java` (メンバーC担当範囲)
```java
package com.example.demo.controller;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.List;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.Accommodation;
import com.example.demo.entity.Plan;
import com.example.demo.entity.PlanStock;
import com.example.demo.repository.AcmCategoryRepository;
import com.example.demo.repository.AcmRepository;
import com.example.demo.repository.PlanRepository;
import com.example.demo.repository.PlanStockRepository;
import com.example.demo.repository.PrefectureRepository;

@Controller
public class AccommodationController {

    private final PrefectureRepository prefectureRepository;
    private final AcmCategoryRepository acmCategoryRepository;
    private final AcmRepository acmRepository;
    private final PlanRepository planRepository;
    private final PlanStockRepository planStockRepository;

    public AccommodationController(PrefectureRepository prefectureRepository, AcmCategoryRepository acmCategoryRepository,
                                   AcmRepository acmRepository, PlanRepository planRepository, PlanStockRepository planStockRepository) {
        this.prefectureRepository = prefectureRepository;
        this.acmCategoryRepository = acmCategoryRepository;
        this.acmRepository = acmRepository;
        this.planRepository = planRepository;
        this.planStockRepository = planStockRepository;
    }

    // 検索・一覧表示 (他メンバー定義の show メソッド名、LocalDate型引数に完全同期)
    @GetMapping("/accommodations")
    public String show(
            @RequestParam(defaultValue = "") String keyword,
            @RequestParam(required = false) Integer prefectureId,
            @RequestParam(required = false) String typeOfAcm,
            @RequestParam(required = false) String checkInTimeStr, // エラー回避のためString受取を推奨
            @RequestParam(required = false) String checkOutTimeStr,
            @RequestParam(defaultValue = "1") Integer peopleCount,
            @RequestParam(defaultValue = "1") Integer roomCount,
            Model model) {

        model.addAttribute("prefectureList", prefectureRepository.findAll());
        model.addAttribute("acmCategoryList", acmCategoryRepository.findAll());
        model.addAttribute("reservableDate", LocalDate.now().plusDays(1));

        // 1. 曖昧検索
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

        // 2. 空室チェック (チェックイン日〜チェックアウト前日までの在庫を確認)
        if (checkInTimeStr != null && !checkInTimeStr.isEmpty() && checkOutTimeStr != null && !checkOutTimeStr.isEmpty()) {
            LocalDate checkIn = LocalDate.parse(checkInTimeStr);
            LocalDate checkOut = LocalDate.parse(checkOutTimeStr);
            LocalDate lastNight = checkOut.minusDays(1);
            long stayDays = java.time.temporal.ChronoUnit.DAYS.between(checkIn, checkOut);

            for (Accommodation hotel : rawHotels) {
                List<Plan> rawPlans = planRepository.findByAccommodationId(hotel.getId());
                boolean hotelHasAvailablePlan = false;

                for (Plan plan : rawPlans) {
                    if (plan.getPeopleCount() < peopleCount) continue;

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

    // 詳細表示 (他メンバー定義の detail メソッド名とURLスキームに同期)
    @GetMapping("/accommodations/{acmId}")
    public String detail(@PathVariable Integer acmId, Model model) {
        Accommodation acm = acmRepository.findById(acmId).orElseThrow();
        List<Plan> planList = planRepository.findByAccommodationId(acmId);

        model.addAttribute("acm", acm);
        model.addAttribute("planList", planList);
        return "accommodation/detail";
    }
}
```

#### `ReservationController.java` (メンバーC担当範囲)
```java
package com.example.demo.controller;

import java.time.LocalDate;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.Plan;
import com.example.demo.entity.User;
import com.example.demo.entity.Reservation;
import com.example.demo.service.ReservationService;
import com.example.demo.repository.PlanRepository;
import com.example.demo.repository.ReservationRepository;

@Controller
public class ReservationController {

    private final ReservationRepository reservationRepository;
    private final PlanRepository planRepository;
    private final ReservationService reservationService;

    public ReservationController(ReservationRepository reservationRepository, PlanRepository planRepository, ReservationService reservationService) {
        this.reservationRepository = reservationRepository;
        this.planRepository = planRepository;
        this.reservationService = reservationService;
    }

    // 予約入力画面表示 (他メンバー定義の showPlanDetail メソッド名とURLスキームに同期)
    @GetMapping("/reservations/{planId}")
    public String showPlanDetail(@PathVariable Integer planId, HttpSession session, Model model) {
        if (session.getAttribute("loginUser") == null) {
            return "redirect:/login";
        }
        Plan plan = planRepository.findById(planId).orElseThrow();
        model.addAttribute("plan", plan);
        return "reserve/new";
    }

    // 予約確認画面 (他メンバー定義の confirmRes メソッド名とLocalDate型引数に同期)
    @GetMapping("/reservations/confirm")
    public String confirmRes(
            @RequestParam Integer planId,
            @RequestParam String checkInDateStr,
            @RequestParam String checkOutDateStr,
            @RequestParam Integer peopleCount,
            @RequestParam(defaultValue = "1") Integer roomCount,
            HttpSession session,
            Model model) {

        if (session.getAttribute("loginUser") == null) {
            return "redirect:/login";
        }

        LocalDate checkIn = LocalDate.parse(checkInDateStr);
        LocalDate checkOut = LocalDate.parse(checkOutDateStr);
        long stayDays = java.time.temporal.ChronoUnit.DAYS.between(checkIn, checkOut);

        Plan plan = planRepository.findById(planId).orElseThrow();
        int totalPrice = (int) (plan.getTaxIncludePrice() * roomCount * stayDays);

        model.addAttribute("plan", plan);
        model.addAttribute("checkInDate", checkIn);
        model.addAttribute("checkOutDate", checkOut);
        model.addAttribute("peopleCount", peopleCount);
        model.addAttribute("roomCount", roomCount);
        model.addAttribute("totalPrice", totalPrice);

        return "reserve/confirm";
    }

    // 予約登録処理 (修正: roomCount引数を追加し、ReservationServiceと連携)
    @PostMapping("/reservations/confirm")
    public String register(
            @RequestParam Integer planId,
            @RequestParam Integer userId,
            @RequestParam String checkInDateStr,
            @RequestParam String checkOutDateStr,
            @RequestParam Integer peopleCount,
            @RequestParam(defaultValue = "1") Integer roomCount,
            HttpSession session,
            Model model) {

        if (session.getAttribute("loginUser") == null) {
            return "redirect:/login";
        }

        LocalDate checkIn = LocalDate.parse(checkInDateStr);
        LocalDate checkOut = LocalDate.parse(checkOutDateStr);

        boolean success = reservationService.processReservation(userId, planId, checkIn, checkOut, peopleCount, roomCount);

        if (!success) {
            Plan plan = planRepository.findById(planId).orElseThrow();
            model.addAttribute("plan", plan);
            model.addAttribute("checkInDate", checkIn);
            model.addAttribute("checkOutDate", checkOut);
            model.addAttribute("peopleCount", peopleCount);
            model.addAttribute("roomCount", roomCount);
            model.addAttribute("message", "空室が足りないため予約できませんでした。");
            return "reserve/confirm";
        }

        return "redirect:/mypage";
    }

    // 会員側・履歴詳細画面表示 (他メンバー定義の showResDetail メソッド名とURLスキームに同期)
    @GetMapping("/users/{userId}/reservations/{resId}")
    public String showResDetail(@PathVariable Integer userId, @PathVariable Integer resId, HttpSession session, Model model) {
        if (session.getAttribute("loginUser") == null) {
            return "redirect:/login";
        }
        Reservation reservation = reservationRepository.findById(resId).orElseThrow();
        model.addAttribute("reservation", reservation);
        return "user/reservation_detail";
    }
}
```

---

## 6. 管理者側コントローラー完全同期コード

#### `AdminAcmController.java` (メンバーD担当範囲: AdminHotelController から同期)
```java
package com.example.demo.controller;

import java.util.List;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.Accommodation;
import com.example.demo.entity.Prefecture;
import com.example.demo.entity.AcmCategory;
import com.example.demo.entity.Plan;
import com.example.demo.repository.AcmRepository;
import com.example.demo.repository.PrefectureRepository;
import com.example.demo.repository.AcmCategoryRepository;
import com.example.demo.repository.PlanRepository;

@Controller
public class AdminAcmController {

    private final AcmRepository acmRepository;
    private final PrefectureRepository prefectureRepository;
    private final AcmCategoryRepository acmCategoryRepository;
    private final PlanRepository planRepository;

    public AdminAcmController(AcmRepository acmRepository, PrefectureRepository prefectureRepository,
                              AcmCategoryRepository acmCategoryRepository, PlanRepository planRepository) {
        this.acmRepository = acmRepository;
        this.prefectureRepository = prefectureRepository;
        this.acmCategoryRepository = acmCategoryRepository;
        this.planRepository = planRepository;
    }

    // 宿一覧 (他メンバー定義の show メソッドと都道府県・カテゴリの絞り込み機能に同期)
    @GetMapping("/admin/accommodations")
    public String show(
            @RequestParam(defaultValue = "0") Integer prefectureId,
            @RequestParam(defaultValue = "") String acmName,
            HttpSession session,
            Model model) {
        
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        model.addAttribute("preList", prefectureRepository.findAll());

        List<Accommodation> acmList;
        if (prefectureId == 0 && acmName.isEmpty()) {
            acmList = acmRepository.findAll();
        } else if (prefectureId > 0 && !acmName.isEmpty()) {
            acmList = acmRepository.findByPrefectureIdAndNameContaining(prefectureId, acmName);
        } else if (prefectureId > 0) {
            acmList = acmRepository.findByPrefectureId(prefectureId);
        } else {
            acmList = acmRepository.findByNameContaining(acmName);
        }

        model.addAttribute("acmList", acmList);
        return "admin/acm_list";
    }

    // 宿追加
    @GetMapping("/admin/accommodations/add")
    public String create(HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";
        model.addAttribute("preList", prefectureRepository.findAll());
        model.addAttribute("categoryList", acmCategoryRepository.findAll());
        return "admin/acm_new";
    }

    @PostMapping("/admin/accommodations/add")
    public String store(
            @RequestParam String name,
            @RequestParam String postalCode,
            @RequestParam Integer prefectureId,
            @RequestParam String address,
            @RequestParam String phoneNumber,
            @RequestParam String checkInTime,
            @RequestParam String checkOutTime,
            @RequestParam Integer typeOfAcm,
            HttpSession session) {

        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Prefecture pref = prefectureRepository.findById(prefectureId).orElseThrow();
        AcmCategory cat = acmCategoryRepository.findById(typeOfAcm).orElseThrow();

        Accommodation acm = new Accommodation(pref, name, postalCode, address, phoneNumber, checkInTime, checkOutTime, cat);
        acmRepository.save(acm);

        return "redirect:/admin/accommodations";
    }

    // 宿詳細 (他メンバー定義の detail メソッド名とURLに同期)
    @GetMapping("/admin/accommodations/detail/{acmId}")
    public String detail(@PathVariable Integer acmId, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation acm = acmRepository.findById(acmId).orElseThrow();
        List<Plan> planList = planRepository.findByAccommodationId(acmId);

        model.addAttribute("acm", acm);
        model.addAttribute("planList", planList);
        return "admin/acm_detail";
    }

    // 宿編集 (他メンバー定義の edit メソッド名とURLに同期)
    @GetMapping("/admin/accommodations/edit/{acmId}")
    public String edit(@PathVariable Integer acmId, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation acm = acmRepository.findById(acmId).orElseThrow();
        model.addAttribute("acm", acm);
        model.addAttribute("preList", prefectureRepository.findAll());
        model.addAttribute("categoryList", acmCategoryRepository.findAll());
        return "admin/acm_edit";
    }

    @PostMapping("/admin/accommodations/edit/{acmId}")
    public String update(
            @PathVariable Integer acmId,
            @RequestParam String name,
            @RequestParam String postalCode,
            @RequestParam Integer prefectureId,
            @RequestParam String address,
            @RequestParam String phoneNumber,
            @RequestParam String checkInTime,
            @RequestParam String checkOutTime,
            @RequestParam Integer typeOfAcm,
            HttpSession session) {

        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation acm = acmRepository.findById(acmId).orElseThrow();
        Prefecture pref = prefectureRepository.findById(prefectureId).orElseThrow();
        AcmCategory cat = acmCategoryRepository.findById(typeOfAcm).orElseThrow();

        acm.setName(name);
        acm.setPrefecture(pref);
        acm.setPostalCode(postalCode);
        acm.setAddress(address);
        acm.setPhoneNumber(phoneNumber);
        acm.setCheckInTime(checkInTime);
        acm.setCheckOutTime(checkOutTime);
        acm.setAcmCategory(cat);

        acmRepository.save(acm);
        return "redirect:/admin/accommodations/detail/" + acm.getId();
    }
}
```

#### `AdminPlanController.java` (メンバーD担当範囲: 他メンバーのURLスキーム・引数に同期)
```java
package com.example.demo.controller;

import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.Accommodation;
import com.example.demo.entity.Plan;
import com.example.demo.repository.AcmRepository;
import com.example.demo.repository.PlanRepository;
import com.example.demo.service.PlanService;

@Controller
public class AdminPlanController {

    private final AcmRepository acmRepository;
    private final PlanRepository planRepository;
    private final PlanService planService; // DIでのサービス受け取り

    public AdminPlanController(AcmRepository acmRepository, PlanRepository planRepository, PlanService planService) {
        this.acmRepository = acmRepository;
        this.planRepository = planRepository;
        this.planService = planService;
    }

    // プラン詳細 (他メンバーの detail メソッド名とURLスキームに同期)
    @GetMapping("/plans/detail/{planId}")
    public String detail(@PathVariable Integer planId, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Plan plan = planRepository.findById(planId).orElseThrow();
        model.addAttribute("plan", plan);
        model.addAttribute("acm", plan.getAccommodation());
        return "admin/plan_detail";
    }

    // プラン追加画面 (他メンバーの create メソッド名とURLスキームに同期)
    @GetMapping("/plans/add/{acmId}")
    public String create(@PathVariable Integer acmId, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation acm = acmRepository.findById(acmId).orElseThrow();
        model.addAttribute("acm", acm);
        return "admin/plan_new";
    }

    // プラン登録実行 (修正: PlanServiceを呼び出し、90日在庫の自動生成処理を実行)
    @PostMapping("/plans/add/{acmId}")
    public String store(
            @PathVariable Integer acmId,
            @RequestParam String name,
            @RequestParam String content,
            @RequestParam Integer taxExcludePrice,
            @RequestParam Integer roomCount,
            @RequestParam Integer peopleCount,
            @RequestParam Boolean isBreakfast,
            HttpSession session) {

        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation accommodation = acmRepository.findById(acmId).orElseThrow();
        
        // 税込金額の算出とプランインスタンス生成 (isArchieved=false)
        int taxIncludePrice = (int) (taxExcludePrice * 1.1);
        Plan plan = new Plan(accommodation, name, content, taxExcludePrice, taxIncludePrice, roomCount, peopleCount, false, isBreakfast);

        // リーダー作成のサービスを呼び出し
        planService.createNewPlan(plan, roomCount);

        return "redirect:/admin/accommodations/detail/" + acmId;
    }
}
```

#### `AdminResController.java` (メンバーE担当範囲: 他メンバーのURLスキーム・引数に同期)
```java
package com.example.demo.controller;

import java.util.List;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.Reservation;
import com.example.demo.entity.User;
import com.example.demo.repository.ReservationRepository;
import com.example.demo.repository.UserRepository;
import com.example.demo.service.ReservationService;

@Controller
public class AdminResController {

    private final ReservationRepository reservationRepository;
    private final UserRepository userRepository;
    private final ReservationService reservationService; // DIでのサービス受け取り

    public AdminResController(ReservationRepository reservationRepository, UserRepository userRepository, ReservationService reservationService) {
        this.reservationRepository = reservationRepository;
        this.userRepository = userRepository;
        this.reservationService = reservationService;
    }

    // 予約一覧 (他メンバー定義の listRes メソッドとキーワード検索に同期)
    @GetMapping("/admin/reservations")
    public String listRes(
            @RequestParam(defaultValue = "") String keyword,
            HttpSession session,
            Model model) {
        
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        List<Reservation> reservationList;
        if (!keyword.isEmpty()) {
            // キーワードに合致する会員に紐づく予約履歴を抽出
            List<User> users = userRepository.findByFamilyNameContainingOrGivenNameContaining(keyword, keyword);
            reservationList = reservationRepository.findAll().stream()
                    .filter(res -> users.stream().anyMatch(u -> u.getId().equals(res.getUser().getId())))
                    .toList();
        } else {
            reservationList = reservationRepository.findAll();
        }

        model.addAttribute("keyword", keyword);
        model.addAttribute("reservations", reservationList);
        return "admin/reservationList";
    }

    // 予約詳細 (修正: DBからの予約取得とバインドを適用)
    @GetMapping("/admin/reservations/detail/{resId}")
    public String showResDetail(@PathVariable Integer resId, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Reservation reservation = reservationRepository.findById(resId)
                .orElseThrow(() -> new IllegalArgumentException("Invalid reservation id: " + resId));
        
        model.addAttribute("reservation", reservation);
        return "admin/reservation_detail";
    }

    // 代理キャンセル (修正: ReservationServiceを呼び出して在庫を復元)
    @PostMapping("/admin/reservations/archive/{resId}")
    public String archiveRes(@PathVariable int resId, HttpSession session) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        reservationService.cancelReservationByAdmin(resId);

        return "redirect:/admin/reservations"; // 正しい予約一覧URLへリダイレクト
    }
}
```

#### `AdminUserController.java` (メンバーE担当範囲: 他メンバーのURLスキーム・引数に同期)
```java
package com.example.demo.controller;

import java.util.List;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import com.example.demo.entity.User;
import com.example.demo.repository.UserRepository;

@Controller
public class AdminUserController {

    private final UserRepository userRepository;

    public AdminUserController(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // 会員一覧
    @GetMapping("/admin/users")
    public String show(HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        List<User> userList = userRepository.findAll();
        model.addAttribute("users", userList);
        return "admin/user_list";
    }

    // 会員詳細 (他メンバー定義の detail メソッド名とURLに同期)
    @GetMapping("/admin/users/{userId}")
    public String detail(@PathVariable Integer userId, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        User user = userRepository.findById(userId)
                .orElseThrow(() -> new IllegalArgumentException("Invalid user id: " + userId));
        
        model.addAttribute("user", user);
        return "admin/user_detail";
    }
}
```

---

## 7. Thymeleaf HTML連携・バインド対応表
ControllerとHTMLテンプレートでやり取りする、最新化されたパラメータの一致関係です。

### 7.1 一般ユーザー側
| 画面ID | 画面名 | パス | 送信・受信パラメータ (name属性) | Model追加キー |
| :--- | :--- | :--- | :--- | :--- |
| **UG101** | 会員新規登録 | `/users/add` | `familyName`, `givenName`, `email`, `postalCode`, `prefectureId`, `address`, `phoneNumber`, `birthDate`, `password` | `pref` (都道府県リスト) |
| **UG102** | ログイン | `/login` | `email`, `password` | `message` (認証エラーメッセージ) |
| **UG103** | マイページ | `/mypage` | 退会POST: `/users/deactivate` (POST) | `user`, `reservations` |
| **UG201** | 宿検索・一覧 | `/accommodations` | `keyword`, `prefectureId`, `typeOfAcm`, `checkInTimeStr`, `checkOutTimeStr`, `peopleCount`, `roomCount` | `prefectureList`, `acmCategoryList`, `acmList` |
| **UG202** | 宿詳細 | `/accommodations/{acmId}` | 次画面遷移: `/reservations/{planId}` | `acm`, `planList` |
| **UG301** | 予約登録入力 | `/reservations/{planId}` | `planId` (Hidden), `checkInDateStr`, `checkOutDateStr`, `peopleCount`, `roomCount` | `plan` |
| **UG302** | 予約内容確認 | `/reservations/confirm` | 送信先: `/reservations/confirm` (POST)<br>引数: `planId`, `userId`, `checkInDateStr`, `checkOutDateStr`, `peopleCount`, `roomCount` | `plan`, `checkInDate`, `checkOutDate`, `peopleCount`, `roomCount`, `totalPrice`, `message` |

### 7.2 管理者側
| 画面ID | 画面名 | パス | 送信・受信パラメータ | Model追加キー |
| :--- | :--- | :--- | :--- | :--- |
| **AG101** | 管理者ログイン | `/admin/login` | `loginId`, `password` | `errorMessage` |
| **AG201** | 宿一覧 | `/admin/accommodations` | `prefectureId`, `acmName` | `preList` (都道府県リスト), `acmList` |
| **AG202** | 宿登録 | `/admin/accommodations/add` | `name`, `postalCode`, `prefectureId`, `address`, `phoneNumber`, `checkInTime`, `checkOutTime`, `typeOfAcm` | `preList`, `categoryList` |
| **AG203** | 宿編集 | `/admin/accommodations/edit/{acmId}`| 上記と同様のパラメータを更新（POSTは`edit/{acmId}`）| `acm`, `preList`, `categoryList` |
| **AG204** | 宿詳細 | `/admin/accommodations/detail/{acmId}`| 次画面遷移: `/plans/add/{acmId}` | `acm`, `planList` |
| **AG301** | プラン詳細 | `/plans/detail/{planId}` | 特になし | `plan`, `acm` |
| **AG302** | プラン登録 | `/plans/add/{acmId}` | `name`, `content`, `taxExcludePrice`, `roomCount`, `peopleCount`, `isBreakfast` | `acm` |
| **AG401** | 会員一覧 | `/admin/users` | 特になし | `users` |
| **AG402** | 会員詳細 | `/admin/users/{userId}` | 特になし | `user` |
| **AG501** | 予約一覧 | `/admin/reservations` | `keyword`（会員名部分一致検索） | `keyword`, `reservations` |
| **AG502** | 予約詳細 | `/admin/reservations/detail/{resId}` | キャンセルPOST: `/reservations/archive/{resId}` | `reservation` |
