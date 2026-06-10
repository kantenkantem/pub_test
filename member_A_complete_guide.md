# メンバーA (テクニカルリーダー) 専用：宿予約システム完全実装マニュアル

本ドキュメントは、チーム開発演習においてテクニカルリーダー（メンバーA）が、自分自身の担当範囲（サービス・在庫管理）を完璧に実装しつつ、**他のメンバー（B〜E）が実装で詰まった際にもコードレベルで即座に解決策を提示・代行できる**ようにするための「全ソースコード・トラブルシューティング集」です。

現在のプロジェクトに定義されている Entity のプロパティや型（`PlanStock` の `planId` がオブジェクトではなく `Integer` である点、`Plan` 内のスペルミス `taxExcludePrice` や `isArchieved` など）に完全準拠しています。

---

## 目次
1. [リポジトリ層の定義 (全Repositoryコード)](#1-リポジトリ層の定義-全repositoryコード)
2. [サービス層の定義 (メンバーA担当：PlanService / ReservationService)](#2-サービス層の定義-メンバーa担当planservice--reservationservice)
3. [会員側コントローラー完全コード (UserController / MypageController)](#3-会員側コントローラー完全コード-usercontroller--mypagecontroller)
4. [会員側検索・予約コントローラー完全コード (AccommodationController / ReservationController)](#4-会員側検索予約コントローラー完全コード-accommodationcontroller--reservationcontroller)
5. [管理者側コントローラー完全コード (AdminHotelController / AdminPlanController / AdminResController / AdminUserController / AdminController)](#5-管理者側コントローラー完全コード)
6. [Thymeleaf HTML連携・バインド対応表](#6-thymeleaf-html連携バインド対応表)

---

## 1. リポジトリ層の定義 (全Repositoryコード)
JPAで自動生成される標準のクエリメソッドを、各インターフェースに定義します。

#### [UserRepository.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/repository/UserRepository.java)
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.User;
import java.util.Optional;

public interface UserRepository extends JpaRepository<User, Integer> {
    // ログインおよびメール重複チェック用
    Optional<User> findByEmail(String email);
    Optional<User> findByEmailAndPassword(String email, String password);
}
```

#### [AcmRepository.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/repository/AcmRepository.java)
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.Accommodation;
import java.util.List;

public interface AcmRepository extends JpaRepository<Accommodation, Integer> {
    // 都道府県のみ指定の検索
    List<Accommodation> findByPrefectureId(Integer prefectureId);
    
    // キーワードのみ指定の曖昧検索
    List<Accommodation> findByNameContaining(String name);
    
    // 都道府県 ＋ キーワード指定の曖昧検索
    List<Accommodation> findByPrefectureIdAndNameContaining(Integer prefectureId, String name);
}
```

#### [PlanRepository.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/repository/PlanRepository.java)
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.Plan;
import java.util.List;

public interface PlanRepository extends JpaRepository<Plan, Integer> {
    // 宿に紐づくすべてのプランを取得（管理者用）
    List<Plan> findByAccommodationId(Integer accommodationId);
    
    // 宿に紐づくアクティブ（アーカイブされていない）プランを取得（一般ユーザー用）
    // Entityのプロパティ名 isArchieved (スペルミスに注意) に準拠
    List<Plan> findByAccommodationIdAndIsArchievedFalse(Integer accommodationId);
}
```

#### [PlanStockRepository.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/repository/PlanStockRepository.java)
```java
package com.example.demo.repository;

import java.time.LocalDate;
import java.util.List;
import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.PlanStock;

public interface PlanStockRepository extends JpaRepository<PlanStock, Integer> {
    // 期間内の在庫レコードを全取得
    List<PlanStock> findByPlanIdAndTargetDateBetween(Integer planId, LocalDate start, LocalDate end);
    
    // 特定の日の在庫を取得・更新用
    List<PlanStock> findByPlanIdAndTargetDate(Integer planId, LocalDate targetDate);
    
    // プラン削除時の在庫全削除用
    void deleteByPlanId(Integer planId);
}
```

#### [ReservationRepository.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/repository/ReservationRepository.java)
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.Reservation;
import java.util.List;

public interface ReservationRepository extends JpaRepository<Reservation, Integer> {
    // 会員の予約履歴を最新順で取得
    List<Reservation> findByUserIdOrderByReservedAtDesc(Integer userId);
    
    // 全予約を最新順で取得（管理者用）
    List<Reservation> findAllByOrderByReservedAtDesc();
}
```

#### [AdminRepository.java](file:///c:/Users/grori/Documents/TDE/team_dev_E_yadoru/src/main/java/com/example/demo/repository/AdminRepository.java)
```java
package com.example.demo.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import com.example.demo.entity.Admin;
import java.util.Optional;

public interface AdminRepository extends JpaRepository<Admin, Integer> {
    Optional<Admin> findByLoginId(String loginId);
}
```

---

## 2. サービス層の定義 (メンバーA担当：PlanService / ReservationService)
トランザクション制御が必要なビジネスロジックをカプセル化します。

#### `PlanService.java` (新規作成)
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

    /**
     * 新規プランを登録し、同時に本日を起点とした180日分の在庫レコード(PlanStock)を自動生成します。
     */
    @Transactional
    public void createNewPlan(Plan plan, Integer initialStock) {
        // 1. プランの保存
        Plan savedPlan = planRepository.save(plan);

        // 2. 180日分の在庫レコードを生成
        LocalDate today = LocalDate.now();
        for (int i = 0; i < 180; i++) {
            PlanStock stock = new PlanStock();
            stock.setPlanId(savedPlan.getId()); // PlanStock内はInteger型
            stock.setTargetDate(today.plusDays(i));
            stock.setStock(initialStock);
            planStockRepository.save(stock);
        }
    }
}
```

#### `ReservationService.java` (新規作成)
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

    /**
     * 在庫チェックと減算、予約レコード登録を同一トランザクション内で行います。
     */
    @Transactional
    public boolean processReservation(Integer userId, Integer planId, LocalDate checkIn, LocalDate checkOut,
                                      Integer peopleCount, Integer roomCount) {
        
        LocalDate lastNight = checkOut.minusDays(1);
        List<PlanStock> stocks = planStockRepository.findByPlanIdAndTargetDateBetween(planId, checkIn, lastNight);

        // 滞在日数分の在庫データがあるか検証
        long stayDays = java.time.temporal.ChronoUnit.DAYS.between(checkIn, checkOut);
        if (stocks.size() != stayDays) {
            return false; // 在庫レコードが存在しない日がある
        }

        // すべての日で部屋数が確保できるか直前チェック
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
        return true;
    }

    /**
     * 管理者が代理でキャンセル処理を行い、在庫を復元します。
     */
    @Transactional
    public void cancelReservationByAdmin(Integer reservationId) {
        Reservation res = reservationRepository.findById(reservationId)
                .orElseThrow(() -> new IllegalArgumentException("Reservation not found: " + reservationId));

        // 二重キャンセル防止
        if (res.getIsCancelled()) {
            return;
        }

        // 1. キャンセルフラグ更新
        res.setIsCancelled(true);
        reservationRepository.save(res);

        // 2. 在庫復元 (チェックイン日〜アウト前日)
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

## 3. 会員側コントローラー完全コード (UserController / MypageController)
LombokやFormクラスは使用せず、`@RequestParam` で個別に引数を受け取って永続化します。

#### `UserController.java` (メンバーB担当)
```java
package com.example.demo.controller;

import java.time.LocalDate;
import java.util.Optional;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.User;
import com.example.demo.repository.UserRepository;

@Controller
public class UserController {

    private final UserRepository userRepository;

    public UserController(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    // ログイン画面表示
    @GetMapping("/login")
    public String showLoginForm() {
        return "user/login";
    }

    // ログイン処理
    @PostMapping("/login")
    public String login(
            @RequestParam("email") String email,
            @RequestParam("password") String password,
            HttpSession session,
            Model model) {
        
        Optional<User> userOpt = userRepository.findByEmailAndPassword(email, password);
        
        if (userOpt.isPresent()) {
            User user = userOpt.get();
            // 退会済みユーザーのログイン阻止
            if (user.getDeactivatedAt() != null) {
                model.addAttribute("errorMessage", "このアカウントは退会済みです。");
                return "user/login";
            }
            session.setAttribute("loginUser", user);
            return "redirect:/accommodations"; // 宿検索画面へ
        } else {
            model.addAttribute("errorMessage", "メールアドレスまたはパスワードが正しくありません。");
            return "user/login";
        }
    }

    // ログアウト処理
    @GetMapping("/logout")
    public String logout(HttpSession session) {
        session.removeAttribute("loginUser");
        return "redirect:/login";
    }

    // 会員登録画面表示
    @GetMapping("/register")
    public String showRegisterForm() {
        return "user/register";
    }

    // 会員登録処理
    @PostMapping("/register")
    public String register(
            @RequestParam("familyName") String familyName,
            @RequestParam("givenName") String givenName,
            @RequestParam("email") String email,
            @RequestParam("password") String password,
            @RequestParam("postalCode") Integer postalCode,
            @RequestParam("address") String address,
            @RequestParam("phoneNumber") String phoneNumber,
            @RequestParam("birthDate") String birthDateStr,
            Model model) {

        // バリデーションチェック (簡易)
        if (userRepository.findByEmail(email).isPresent()) {
            model.addAttribute("errorMessage", "このメールアドレスは既に登録されています。");
            return "user/register";
        }

        User user = new User();
        user.setFamilyName(familyName);
        user.setGivenName(givenName);
        user.setEmail(email);
        user.setPassword(password); // 演習のため平文
        user.setPostalCode(postalCode);
        user.setAddress(address);
        user.setPhoneNumber(phoneNumber);
        user.setBirthDate(LocalDate.parse(birthDateStr));
        user.setRegisteredAt(LocalDate.now());

        userRepository.save(user);
        return "redirect:/login";
    }
}
```

#### `MypageController.java` (メンバーB担当)
```java
package com.example.demo.controller;

import java.time.LocalDate;
import java.util.List;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.User;
import com.example.demo.entity.Reservation;
import com.example.demo.repository.UserRepository;
import com.example.demo.repository.ReservationRepository;

@Controller
public class MypageController {

    private final UserRepository userRepository;
    private final ReservationRepository reservationRepository;

    public MypageController(UserRepository userRepository, ReservationRepository reservationRepository) {
        this.userRepository = userRepository;
        this.reservationRepository = reservationRepository;
    }

    // マイページ（予約履歴一覧）
    @GetMapping("/mypage")
    public String showMypage(HttpSession session, Model model) {
        User loginUser = (User) session.getAttribute("loginUser");
        if (loginUser == null) {
            return "redirect:/login";
        }

        // 最新のユーザー情報をDBから再取得
        User user = userRepository.findById(loginUser.getId()).orElse(loginUser);
        
        List<Reservation> reservations = reservationRepository.findByUserIdOrderByReservedAtDesc(user.getId());
        model.addAttribute("user", user);
        model.addAttribute("reservations", reservations);
        
        return "user/mypage";
    }

    // 会員退会処理（論理削除）
    @PostMapping("/mypage/deactivate")
    public String deactivateUser(HttpSession session) {
        User loginUser = (User) session.getAttribute("loginUser");
        if (loginUser == null) {
            return "redirect:/login";
        }

        User user = userRepository.findById(loginUser.getId()).orElseThrow();
        user.setDeactivatedAt(LocalDate.now()); // 退会日付を設定
        userRepository.save(user);

        session.removeAttribute("loginUser"); // セッションクリア
        return "redirect:/login";
    }
}
```

---

## 4. 会員側検索・予約コントローラー完全コード (AccommodationController / ReservationController)
Entityの構造を変更せず、**JavaのMapを用いて「宿とそれに紐づくプラン」を画面に送る**ことでコンフリクトや実行エラーを完全に排除しています。

#### `AccommodationController.java` (メンバーC担当)
```java
package com.example.demo.controller;

import java.time.LocalDate;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.Accommodation;
import com.example.demo.entity.Plan;
import com.example.demo.entity.PlanStock;
import com.example.demo.entity.Prefecture;
import com.example.demo.repository.AcmRepository;
import com.example.demo.repository.PlanRepository;
import com.example.demo.repository.PlanStockRepository;
import com.example.demo.repository.PrefectureRepository;

@Controller
public class AccommodationController {

    private final AcmRepository acmRepository;
    private final PlanRepository planRepository;
    private final PlanStockRepository planStockRepository;
    private final PrefectureRepository prefectureRepository;

    public AccommodationController(AcmRepository acmRepository, PlanRepository planRepository,
                                   PlanStockRepository planStockRepository, PrefectureRepository prefectureRepository) {
        this.acmRepository = acmRepository;
        this.planRepository = planRepository;
        this.planStockRepository = planStockRepository;
        this.prefectureRepository = prefectureRepository;
    }

    // 宿検索画面
    @GetMapping("/accommodations")
    public String searchAccommodations(
            @RequestParam(value = "prefectureId", required = false) Integer prefectureId,
            @RequestParam(value = "keyword", required = false) String keyword,
            @RequestParam(value = "checkInDate", required = false) String checkInDateStr,
            @RequestParam(value = "checkOutDate", required = false) String checkOutDateStr,
            @RequestParam(value = "peopleCount", defaultValue = "1") Integer peopleCount,
            @RequestParam(value = "roomCount", defaultValue = "1") Integer roomCount,
            Model model) {

        // 都道府県マスタを渡す (プルダウン用)
        List<Prefecture> prefectures = prefectureRepository.findAll();
        model.addAttribute("prefectures", prefectures);

        // 1. 宿情報の曖昧検索 (JPA)
        List<Accommodation> rawHotels;
        if (prefectureId != null && keyword != null && !keyword.isEmpty()) {
            rawHotels = acmRepository.findByPrefectureIdAndNameContaining(prefectureId, keyword);
        } else if (prefectureId != null) {
            rawHotels = acmRepository.findByPrefectureId(prefectureId);
        } else if (keyword != null && !keyword.isEmpty()) {
            rawHotels = acmRepository.findByNameContaining(keyword);
        } else {
            rawHotels = acmRepository.findAll();
        }

        List<Accommodation> resultHotels = new ArrayList<>();
        // 宿ID => 絞り込み後の有効プランのMap (Entity改変回避のため)
        Map<Integer, List<Plan>> hotelPlansMap = new HashMap<>();

        // 日付入力がある場合のみ空室検索フィルタリングを実行
        if (checkInDateStr != null && !checkInDateStr.isEmpty() && checkOutDateStr != null && !checkOutDateStr.isEmpty()) {
            LocalDate checkIn = LocalDate.parse(checkInDateStr);
            LocalDate checkOut = LocalDate.parse(checkOutDateStr);
            LocalDate lastNight = checkOut.minusDays(1);
            long stayDays = java.time.temporal.ChronoUnit.DAYS.between(checkIn, checkOut);

            for (Accommodation hotel : rawHotels) {
                // アーカイブされていないアクティブプランのみ取得
                List<Plan> rawPlans = planRepository.findByAccommodationIdAndIsArchievedFalse(hotel.getId());
                List<Plan> validPlans = new ArrayList<>();

                for (Plan plan : rawPlans) {
                    // 人数上限チェック
                    if (plan.getPeopleCount() < peopleCount) {
                        continue;
                    }

                    // 期間内の在庫確認
                    List<PlanStock> stocks = planStockRepository.findByPlanIdAndTargetDateBetween(plan.getId(), checkIn, lastNight);
                    boolean isAvailable = (stocks.size() == stayDays);

                    for (PlanStock stock : stocks) {
                        if (stock.getStock() < roomCount) {
                            isAvailable = false;
                            break;
                        }
                    }

                    if (isAvailable) {
                        validPlans.add(plan);
                    }
                }

                // 1つ以上有効なプランがある宿のみ検索結果に加える
                if (!validPlans.isEmpty()) {
                    resultHotels.add(hotel);
                    hotelPlansMap.put(hotel.getId(), validPlans);
                }
            }
        } else {
            resultHotels = rawHotels;
            for (Accommodation hotel : rawHotels) {
                List<Plan> plans = planRepository.findByAccommodationIdAndIsArchievedFalse(hotel.getId());
                hotelPlansMap.put(hotel.getId(), plans);
            }
        }

        model.addAttribute("hotels", resultHotels);
        model.addAttribute("hotelPlansMap", hotelPlansMap);
        return "accommodation/search";
    }

    // 宿詳細（プラン一覧）画面
    @GetMapping("/accommodations/detail")
    public String showHotelDetail(@RequestParam("id") Integer id, Model model) {
        Accommodation hotel = acmRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Invalid hotel id: " + id));
        List<Plan> plans = planRepository.findByAccommodationIdAndIsArchievedFalse(hotel.getId());
        
        model.addAttribute("hotel", hotel);
        model.addAttribute("plans", plans);
        return "accommodation/detail";
    }
}
```

#### `ReservationController.java` (メンバーC担当)
```java
package com.example.demo.controller;

import java.time.LocalDate;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import com.example.demo.entity.User;
import com.example.demo.entity.Plan;
import com.example.demo.service.ReservationService;
import com.example.demo.repository.PlanRepository;

@Controller
public class ReservationController {

    private final PlanRepository planRepository;
    private final ReservationService reservationService;

    public ReservationController(PlanRepository planRepository, ReservationService reservationService) {
        this.planRepository = planRepository;
        this.reservationService = reservationService;
    }

    // 予約情報入力・確認画面への遷移
    @GetMapping("/reservations/new")
    public String showReservationForm(
            @RequestParam("planId") Integer planId,
            HttpSession session,
            Model model) {
        
        User loginUser = (User) session.getAttribute("loginUser");
        if (loginUser == null) {
            return "redirect:/login";
        }

        Plan plan = planRepository.findById(planId)
                .orElseThrow(() -> new IllegalArgumentException("Invalid plan id: " + planId));
        
        model.addAttribute("plan", plan);
        return "reserve/new";
    }

    // 予約内容確認画面の表示
    @PostMapping("/reservations/confirm")
    public String confirmReservation(
            @RequestParam("planId") Integer planId,
            @RequestParam("checkInDate") String checkInDate,
            @RequestParam("checkOutDate") String checkOutDate,
            @RequestParam("peopleCount") Integer peopleCount,
            @RequestParam("roomCount") Integer roomCount,
            HttpSession session,
            Model model) {

        if (session.getAttribute("loginUser") == null) {
            return "redirect:/login";
        }

        Plan plan = planRepository.findById(planId).orElseThrow();
        
        // 画面確認用の属性追加
        model.addAttribute("plan", plan);
        model.addAttribute("checkInDate", checkInDate);
        model.addAttribute("checkOutDate", checkOutDate);
        model.addAttribute("peopleCount", peopleCount);
        model.addAttribute("roomCount", roomCount);
        
        // 金額の計算（一泊あたりの金額 × 部屋数 × 宿泊日数）
        LocalDate checkIn = LocalDate.parse(checkInDate);
        LocalDate checkOut = LocalDate.parse(checkOutDate);
        long stayDays = java.time.temporal.ChronoUnit.DAYS.between(checkIn, checkOut);
        
        // Entityのフィールド taxIncludePrice に準拠
        int totalPrice = (int) (plan.getTaxIncludePrice() * roomCount * stayDays);
        model.addAttribute("totalPrice", totalPrice);

        return "reserve/confirm";
    }

    // 予約の最終確定実行
    @PostMapping("/reservations")
    public String createReservation(
            @RequestParam("planId") Integer planId,
            @RequestParam("checkInDate") String checkInDate,
            @RequestParam("checkOutDate") String checkOutDate,
            @RequestParam("peopleCount") Integer peopleCount,
            @RequestParam("roomCount") Integer roomCount,
            HttpSession session,
            Model model) {

        User loginUser = (User) session.getAttribute("loginUser");
        if (loginUser == null) {
            return "redirect:/login";
        }

        LocalDate checkIn = LocalDate.parse(checkInDate);
        LocalDate checkOut = LocalDate.parse(checkOutDate);

        // トランザクション処理をServiceで実行
        boolean success = reservationService.processReservation(
                loginUser.getId(), planId, checkIn, checkOut, peopleCount, roomCount
        );

        if (!success) {
            Plan plan = planRepository.findById(planId).orElseThrow();
            model.addAttribute("plan", plan);
            model.addAttribute("checkInDate", checkInDate);
            model.addAttribute("checkOutDate", checkOutDate);
            model.addAttribute("peopleCount", peopleCount);
            model.addAttribute("roomCount", roomCount);
            model.addAttribute("errorMessage", "タッチの差で満室になってしまったため、予約が確定できませんでした。");
            return "reserve/confirm"; // 確認画面へ戻してエラー表示
        }

        return "redirect:/mypage";
    }
}
```

---

## 5. 管理者側コントローラー完全コード

#### `AdminController.java` (共通)
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

    @GetMapping("/admin/login")
    public String showLoginForm() {
        return "admin/login";
    }

    @PostMapping("/admin/login")
    public String login(
            @RequestParam("loginId") String loginId,
            @RequestParam("password") String password,
            HttpSession session,
            Model model) {

        Optional<Admin> adminOpt = adminRepository.findByLoginId(loginId);

        if (adminOpt.isPresent() && adminOpt.get().getPassword().equals(password)) {
            session.setAttribute("loginAdmin", adminOpt.get());
            return "redirect:/admin/hotels"; // 宿一覧画面へ
        } else {
            model.addAttribute("errorMessage", "管理者IDまたはパスワードが正しくありません。");
            return "admin/login";
        }
    }

    @GetMapping("/admin/logout")
    public String logout(HttpSession session) {
        session.removeAttribute("loginAdmin");
        return "redirect:/admin/login";
    }
}
```

#### `AdminHotelController.java` (メンバーD担当)
```java
package com.example.demo.controller;

import java.util.List;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.PathVariable;
import com.example.demo.entity.Accommodation;
import com.example.demo.entity.Prefecture;
import com.example.demo.entity.AcmCategory;
import com.example.demo.repository.AcmRepository;
import com.example.demo.repository.PrefectureRepository;
import com.example.demo.repository.AcmCategoryRepository;

@Controller
public class AdminHotelController {

    private final AcmRepository acmRepository;
    private final PrefectureRepository prefectureRepository;
    private final AcmCategoryRepository acmCategoryRepository;

    public AdminHotelController(AcmRepository acmRepository, PrefectureRepository prefectureRepository,
                                AcmCategoryRepository acmCategoryRepository) {
        this.acmRepository = acmRepository;
        this.prefectureRepository = prefectureRepository;
        this.acmCategoryRepository = acmCategoryRepository;
    }

    // 宿一覧
    @GetMapping("/admin/hotels")
    public String listHotels(HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";
        
        List<Accommodation> hotels = acmRepository.findAll();
        model.addAttribute("hotels", hotels);
        return "admin/hotel_list";
    }

    // 宿追加画面
    @GetMapping("/admin/hotels/new")
    public String showNewForm(HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        model.addAttribute("prefectures", prefectureRepository.findAll());
        model.addAttribute("categories", acmCategoryRepository.findAll());
        return "admin/hotel_new";
    }

    // 宿登録処理
    @PostMapping("/admin/hotels")
    public String createHotel(
            @RequestParam("name") String name,
            @RequestParam("prefectureId") Integer prefectureId,
            @RequestParam("postalCode") String postalCode,
            @RequestParam("address") String address,
            @RequestParam("phoneNumber") String phoneNumber,
            @RequestParam("checkInTime") String checkInTime,
            @RequestParam("checkOutTime") String checkOutTime,
            @RequestParam("acmCategoryId") Integer acmCategoryId,
            HttpSession session) {

        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Prefecture pref = prefectureRepository.findById(prefectureId).orElseThrow();
        AcmCategory cat = acmCategoryRepository.findById(acmCategoryId).orElseThrow();

        Accommodation hotel = new Accommodation(pref, name, postalCode, address, phoneNumber, checkInTime, checkOutTime, cat);
        acmRepository.save(hotel);

        return "redirect:/admin/hotels";
    }

    // 宿編集画面
    @GetMapping("/admin/hotels/{id}/edit")
    public String showEditForm(@PathVariable("id") Integer id, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation hotel = acmRepository.findById(id).orElseThrow();
        model.addAttribute("hotel", hotel);
        model.addAttribute("prefectures", prefectureRepository.findAll());
        model.addAttribute("categories", acmCategoryRepository.findAll());
        return "admin/hotel_edit";
    }

    // 宿更新処理
    @PostMapping("/admin/hotels/{id}/update")
    public String updateHotel(
            @PathVariable("id") Integer id,
            @RequestParam("name") String name,
            @RequestParam("prefectureId") Integer prefectureId,
            @RequestParam("postalCode") String postalCode,
            @RequestParam("address") String address,
            @RequestParam("phoneNumber") String phoneNumber,
            @RequestParam("checkInTime") String checkInTime,
            @RequestParam("checkOutTime") String checkOutTime,
            @RequestParam("acmCategoryId") Integer acmCategoryId,
            HttpSession session) {

        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation hotel = acmRepository.findById(id).orElseThrow();
        Prefecture pref = prefectureRepository.findById(prefectureId).orElseThrow();
        AcmCategory cat = acmCategoryRepository.findById(acmCategoryId).orElseThrow();

        hotel.setName(name);
        hotel.setPrefecture(pref);
        hotel.setPostalCode(postalCode);
        hotel.setAddress(address);
        hotel.setPhoneNumber(phoneNumber);
        hotel.setCheckInTime(checkInTime);
        hotel.setCheckOutTime(checkOutTime);
        hotel.setAcmCategory(cat);

        acmRepository.save(hotel);
        return "redirect:/admin/hotels";
    }
}
```

#### `AdminPlanController.java` (メンバーD担当)
```java
package com.example.demo.controller;

import java.util.List;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.PathVariable;
import com.example.demo.entity.Accommodation;
import com.example.demo.entity.Plan;
import com.example.demo.repository.AcmRepository;
import com.example.demo.repository.PlanRepository;
import com.example.demo.service.PlanService;

@Controller
public class AdminPlanController {

    private final AcmRepository acmRepository;
    private final PlanRepository planRepository;
    private final PlanService planService; // メンバーA作成のサービスをインジェクション

    public AdminPlanController(AcmRepository acmRepository, PlanRepository planRepository, PlanService planService) {
        this.acmRepository = acmRepository;
        this.planRepository = planRepository;
        this.planService = planService;
    }

    // 宿に紐づくプラン一覧
    @GetMapping("/admin/hotels/{hotelId}/plans")
    public String listPlans(@PathVariable("hotelId") Integer hotelId, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation hotel = acmRepository.findById(hotelId).orElseThrow();
        List<Plan> plans = planRepository.findByAccommodationId(hotelId);

        model.addAttribute("hotel", hotel);
        model.addAttribute("plans", plans);
        return "admin/plan_list";
    }

    // プラン追加画面
    @GetMapping("/admin/hotels/{hotelId}/plans/new")
    public String showNewForm(@PathVariable("hotelId") Integer hotelId, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation hotel = acmRepository.findById(hotelId).orElseThrow();
        model.addAttribute("hotel", hotel);
        return "admin/plan_new";
    }

    // プラン登録処理 (Serviceを呼び出して180日分の在庫レコードも自動生成する)
    @PostMapping("/admin/hotels/{hotelId}/plans")
    public String createPlan(
            @PathVariable("hotelId") Integer hotelId,
            @RequestParam("name") String name,
            @RequestParam("content") String content,
            @RequestParam("taxExcludePrice") Integer taxExcludePrice,
            @RequestParam("taxIncludePrice") Integer taxIncludePrice,
            @RequestParam("roomCount") Integer roomCount,
            @RequestParam("peopleCount") Integer peopleCount,
            @RequestParam("isBreakfast") Boolean isBreakfast,
            HttpSession session) {

        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Accommodation hotel = acmRepository.findById(hotelId).orElseThrow();

        Plan plan = new Plan();
        plan.setAccommodation(hotel);
        plan.setName(name);
        plan.setContent(content);
        plan.setTaxExcludePrice(taxExcludePrice); // Entityのスペル taxExcludePrice に準拠
        plan.setTaxIncludePrice(taxIncludePrice);
        plan.setRoomCount(roomCount);
        plan.setPeopleCount(peopleCount);
        plan.setIsBreakfast(isBreakfast);
        plan.setIsArchieved(false);

        // トランザクション対応サービスで保存 (初期在庫設定はプラン自体の部屋数)
        planService.createNewPlan(plan, roomCount);

        return "redirect:/admin/hotels/" + hotelId + "/plans";
    }

    // プランのアーカイブ処理 (論理削除)
    @PostMapping("/admin/hotels/{hotelId}/plans/{planId}/archive")
    public String archivePlan(
            @PathVariable("hotelId") Integer hotelId,
            @PathVariable("planId") Integer planId,
            HttpSession session) {
        
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        Plan plan = planRepository.findById(planId).orElseThrow();
        plan.setIsArchieved(true); // Entity定義 isArchieved に準拠
        planRepository.save(plan);

        return "redirect:/admin/hotels/" + hotelId + "/plans";
    }
}
```

#### `AdminResController.java` (メンバーE担当)
```java
package com.example.demo.controller;

import java.util.List;
import jakarta.servlet.http.HttpSession;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PathVariable;
import com.example.demo.entity.Reservation;
import com.example.demo.repository.ReservationRepository;
import com.example.demo.service.ReservationService;

@Controller
public class AdminResController {

    private final ReservationRepository reservationRepository;
    private final ReservationService reservationService; // メンバーA作成

    public AdminResController(ReservationRepository reservationRepository, ReservationService reservationService) {
        this.reservationRepository = reservationRepository;
        this.reservationService = reservationService;
    }

    // 予約一覧表示
    @GetMapping("/admin/reservations")
    public String listReservations(HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        List<Reservation> reservations = reservationRepository.findAllByOrderByReservedAtDesc();
        model.addAttribute("reservations", reservations);
        return "admin/reservation_list";
    }

    // 代理キャンセル処理
    @PostMapping("/admin/reservations/{id}/cancel")
    public String cancelReservation(@PathVariable("id") Integer id, HttpSession session) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        // Service経由でキャンセルフラグの変更と在庫の復元を同時に行う
        reservationService.cancelReservationByAdmin(id);

        return "redirect:/admin/reservations";
    }
}
```

#### `AdminUserController.java` (メンバーE担当)
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

    // 会員一覧表示
    @GetMapping("/admin/users")
    public String listUsers(HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        List<User> users = userRepository.findAll();
        model.addAttribute("users", users);
        return "admin/user_list";
    }

    // 会員詳細表示
    @GetMapping("/admin/users/{id}")
    public String showUserDetail(@PathVariable("id") Integer id, HttpSession session, Model model) {
        if (session.getAttribute("loginAdmin") == null) return "redirect:/admin/login";

        User user = userRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("User not found: " + id));
        model.addAttribute("user", user);
        return "admin/user_detail";
    }
}
```

---

## 6. Thymeleaf HTML連携・バインド対応表
ControllerとHTMLテンプレートで受け渡す変数・オブジェクト・およびパラメータの対応関係です。メンバーのコーディング時のデバッグチェックに使えます。

### 6.1 一般ユーザー側
| 画面ID | 画面名 | パス | 送信・受信パラメータ (name属性) | Model追加キー |
| :--- | :--- | :--- | :--- | :--- |
| **UG101** | 新規会員登録 | `/register` | `familyName`, `givenName`, `email`, `password`, `postalCode`, `address`, `phoneNumber`, `birthDate` | `errorMessage` (登録エラー時) |
| **UG102** | ログイン | `/login` | `email`, `password` | `errorMessage` (ログイン失敗時) |
| **UG103** | マイページ | `/mypage` | 退会実行時: `/mypage/deactivate` (POST) | `user`, `reservations` |
| **UG201** | 宿検索・一覧 | `/accommodations` | `prefectureId`, `keyword`, `checkInDate`, `checkOutDate`, `peopleCount`, `roomCount` | `prefectures`, `hotels`, `hotelPlansMap` |
| **UG202** | 宿詳細・プラン | `/accommodations/detail` | `/reservations/new?planId=X` (GETでプランID送る) | `hotel`, `plans` |
| **UG301** | 予約情報入力 | `/reservations/new` | `planId` (Hidden), `checkInDate`, `checkOutDate`, `peopleCount`, `roomCount` | `plan` |
| **UG302** | 予約確認 | `/reservations/confirm` | 送信先: `/reservations` (POST)<br>引数: `planId`, `checkInDate`, `checkOutDate`, `peopleCount`, `roomCount` | `plan`, `checkInDate`, `checkOutDate`, `peopleCount`, `roomCount`, `totalPrice`, `errorMessage` |

### 6.2 管理者側
| 画面ID | 画面名 | パス | 送信・受信パラメータ | Model追加キー |
| :--- | :--- | :--- | :--- | :--- |
| **AG101** | 管理者ログイン | `/admin/login` | `loginId`, `password` | `errorMessage` |
| **AG201** | 宿一覧 | `/admin/hotels` | 特になし | `hotels` |
| **AG202** | 宿登録 | `/admin/hotels/new` | `name`, `prefectureId`, `postalCode`, `address`, `phoneNumber`, `checkInTime`, `checkOutTime`, `acmCategoryId` | `prefectures`, `categories` |
| **AG203** | 宿編集 | `/admin/hotels/{id}/edit` | 上記と同様のパラメータを更新（POSTは`/admin/hotels/{id}/update`） | `hotel`, `prefectures`, `categories` |
| **AG301** | プラン一覧 | `/admin/hotels/{hotelId}/plans` | アーカイブ送信: `{hotelId}/plans/{planId}/archive` (POST) | `hotel`, `plans` |
| **AG302** | プラン登録 | `/admin/hotels/{hotelId}/plans/new` | `name`, `content`, `taxExcludePrice`, `taxIncludePrice`, `roomCount`, `peopleCount`, `isBreakfast` | `hotel` |
| **AG401** | 会員一覧 | `/admin/users` | 特になし | `users` |
| **AG402** | 会員詳細 | `/admin/users/{id}` | 特になし | `user` |
| **AG501** | 予約一覧 | `/admin/reservations` | キャンセル実行: `/admin/reservations/{id}/cancel` (POST) | `reservations` |
