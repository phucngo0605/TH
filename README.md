# README chi tiet: he thong AEB va Lane Keeping

Tai lieu nay mo ta cach he thong phanh khan cap tu dong AEB va ho tro giu lan LKAS trong thu muc `aeb/` dang hoat dong. Noi dung duoc viet theo logic code hien tai, khong phai mo ta chung chung.

## 1. Tong quan he thong

Ung dung chay tren CARLA va dieu khien xe ego bang cac module chinh:

| Thanh phan | File chinh | Vai tro |
|---|---|---|
| Vong lap chinh | `aeb/main.py` | Doc camera/radar, chay YOLO, tinh AEB/LKAS, gui lenh dieu khien xe |
| Camera va YOLO | `aeb/modules/perception.py` | Phat hien 4 class nguy hiem: `person`, `bike_motorbike`, `car`, `truck` |
| Theo doi box on dinh | `aeb/modules/object_lock.py` | Giu track object qua nhieu frame, giam nhap nhay va false positive |
| Radar LRR/SRR | `aeb/modules/radar_sonar.py` | Lay diem radar, loc vat can theo toa do xe va closing speed |
| Hop nhat cam bien | `aeb/modules/sensor_fusion.py` | Quyet dinh co can canh bao/phanh hay khong |
| Bo phanh | `aeb/modules/braking.py` | Giai doan hien tai dung hard brake: co nguy hiem thi phanh 1.0 |
| Dieu khien xe | `aeb/modules/controller.py` | Apply throttle/brake/steer/handbrake len CARLA |
| Lane segmentation | `aeb/modules/lane_runtime.py` | Chay model lane, loc mask, on dinh ket qua lane |
| Lane keeping | `aeb/modules/lane_keeping.py` | Tinh goc lai ho tro dua xe ve giua lan |
| Cau hinh | `aeb/configs/aeb_config.py` | Tap trung nguong radar, AEB, YOLO, LKAS |

Luong xu ly tong quat:

```text
Camera RGB
   -> YOLO detector
   -> StableDetectionTracker
   -> danh sach object on dinh

Radar LRR truoc + SRR trai + SRR phai
   -> loc diem theo khoang cach, lateral, cao do, closing speed
   -> gom cum diem radar thanh threat

Lane camera frame
   -> lane segmentation
   -> loc hinh hoc mask
   -> on dinh lane theo thoi gian
   -> LKAS tinh steer assist

YOLO boxes + radar threats + toc do xe + steering
   -> SensorFusion
   -> FusionResult
   -> BrakingController
   -> VehicleController apply brake/throttle/steer
```

## 2. Cam bien dang dung

### 2.1 Camera truoc

Camera duoc spawn phia truoc xe ego:

| Tham so | Gia tri hien tai |
|---|---:|
| Do phan giai | `1280x720` |
| FOV | `90` do |
| FPS | `20` |
| Vi tri | `x=0.5`, `z=1.3` |

Camera duoc dung cho hai tac vu:

1. Phat hien vat the AEB bang YOLO.
2. Phan doan lane de tao mask lan duong cho LKAS.

### 2.2 YOLO object detector

Model YOLO duoc load tu:

```text
aeb/weights/yolo26_aeb_best.onnx
```

Class duoc su dung:

| Class | Y nghia |
|---|---|
| `person` | Nguoi di bo |
| `bike_motorbike` | Xe dap / xe may |
| `car` | O to |
| `truck` | Xe tai |

Nguong tin cay mac dinh:

```text
CONFIDENCE_THRESHOLD = 0.55
```

Trong `perception.py` co them mot so loc de giam nhan sai:

- Loai bot box `person` yeu hoac hinh dang khong hop ly.
- Loc vat can mau cam/trang kieu rao chan cong trinh bi nham thanh nguoi/xe.
- Dung tracker de box khong nhap nhay tung frame.

Luu y: YOLO chi nhan 4 class tren. Cac vat can khac nhu tuong, cot dien, cay, rao chan, via he... khong phu thuoc vao class YOLO ma chu yeu duoc xu ly bang radar.

### 2.3 Radar LRR phia truoc

LRR la radar tam xa phia truoc xe, dung de phat hien vat can nam tren duong di chinh cua xe.

| Tham so | Gia tri hien tai |
|---|---:|
| Range | `150 m` |
| Horizontal FOV | `16` do |
| Vertical FOV | `10` do |
| Vi tri | `x=2.4`, `y=0.0`, `z=0.8` |
| Huong | `yaw=0` |

LRR duoc dung cho:

- Xe phia truoc.
- Nguoi/xe dap/xe may neu nam trong vung duong di.
- Vat can khong co class YOLO: tuong, cot, rao chan, cay, vat the bat ky neu radar nhin thay va nam trong corridor.

### 2.4 Radar SRR hai ben truoc

Hai SRR nam hai ben mui xe, dung de bat vat can gan, vat can cheo, va truong hop xe danh lai vao tuong/cot.

| Radar | Vi tri | Huong | Range | FOV |
|---|---|---:|---:|---:|
| SRR-L | `x=2.2`, `y=-0.9`, `z=0.6` | `yaw=-45` | `30 m` | `100` do |
| SRR-R | `x=2.2`, `y=0.9`, `z=0.6` | `yaw=45` | `30 m` | `100` do |

SRR duoc dung cho:

- Vat can rat gan truoc mui xe.
- Vat can lech trai/phai khi xe dang danh lai vao.
- Xu ly tinh huong "dang di ma quay lai/danh lai vao tuong, cot dien, cay, vat can".

## 3. Loc radar va toa do radar

Radar CARLA tra ve cac diem detection. Code chuyen moi diem ve toa do gan voi xe ego:

```text
vehicle_x: khoang cach phia truoc xe
vehicle_y: lech trai/phai so voi tam xe
vehicle_z: do cao diem radar
closing_speed: toc do vat the dang tien lai gan xe
```

Mot diem radar chi duoc coi la ung vien nguy hiem neu qua cac bo loc:

| Bo loc | Muc dich |
|---|---|
| Khoang cach toi thieu | Bo diem qua sat/nhieu do CARLA |
| Closing speed | Uu tien diem dang tien lai gan xe |
| Cao do | Bo diem o mat dat hoac qua cao |
| Lateral corridor | Chi lay diem nam trong hanh lang duong di |
| Cluster | Yeu cau nhieu diem gan nhau, tranh 1 diem nhieu lam phanh gia |

Nguong quan trong:

```text
RADAR_MIN_DEPTH = 1.5 m
RADAR_MIN_CLOSING = 1.0 m/s
RADAR_CLUSTER_MIN_POINTS = 3
RADAR_VEHICLE_Z_MIN = 0.12 m
RADAR_VEHICLE_Z_MAX = 2.80 m
```

Voi LRR, hanh lang phia truoc co nua rong co ban:

```text
RADAR_LRR_CORRIDOR_HALF_WIDTH = 1.35 m
```

Va rong them theo khoang cach:

```text
RADAR_LRR_CORRIDOR_GROWTH = 0.012
```

Voi SRR, vung xu ly vat can gan:

```text
RADAR_SRR_COLLISION_HALF_WIDTH = 1.45 m
RADAR_SRR_HARD_BRAKE_DISTANCE_M = 3.0 m
```

## 4. Cach AEB ra quyet dinh

AEB khong chi dua vao box YOLO. Quyet dinh phanh nam trong `SensorFusion.compute()` va chay theo thu tu uu tien sau.

### 4.1 Truong hop 1: collision sensor da bao va cham

Neu collision sensor bao va cham, he thong lap tuc tra ve:

```text
source = COLLISION
reason = collision_emergency
brake_force = 1.0
critical = true
```

Day la lop bao ve cuoi cung, dung de ghi nhan test FAIL/PASS va ep xe dung lai sau va cham. Neu den buoc nay moi phanh thi xem nhu AEB da phan ung qua muon.

### 4.2 Truong hop 2: toc do qua thap

Neu toc do xe nho hon:

```text
AEB_MIN_SPEED_KMH = 5.0 km/h
```

Fusion tra ve:

```text
source = speed_too_low
should_brake = false
```

Muc dich la tranh xe dung yen hoac bo cham bi phanh giu lien tuc do nhieu radar.

### 4.3 Truong hop 3: SRR vat can gan phia truoc hoac hai ben

SRR co uu tien cao hon LRR vi day la vat can rat gan. Neu SRR-L hoac SRR-R phat hien cum diem nguy hiem, he thong co the phanh cung.

Hai kieu nguy hiem SRR:

1. Vat can nam ngay vung truoc mui xe.
2. Vat can ben trai/phai va xe dang danh lai ve phia vat can.

Voi side impact, code xem:

```text
steer_value
vehicle_x
vehicle_y
closing_speed
ttc
```

Neu xe dang danh lai vao tuong/cot/vat can ben hong va vat can nam trong duong di du kien, AEB co the phanh ke ca YOLO khong co box.

Thong so side impact:

```text
AEB_SRR_SIDE_IMPACT_ENABLED = True
AEB_SRR_SIDE_STEER_THRESHOLD = 0.12
AEB_SRR_SIDE_FORWARD_MIN_M = 0.0
AEB_SRR_SIDE_FORWARD_MAX_M = 10.0
AEB_SRR_SIDE_LATERAL_MIN_M = 0.55
AEB_SRR_SIDE_LATERAL_MAX_M = 4.5
AEB_SRR_SIDE_TTC_SECONDS = 0.45
AEB_SRR_SIDE_HARD_LATERAL_DISTANCE_M = 1.0
```

Ket qua thuong gap:

```text
source = SRR-L / SRR-R
reason = srr_near_obstacle hoac srr_side_impact
brake_force = 1.0
critical = true
```

### 4.4 Truong hop 4: LRR + camera confirm

Day la truong hop tot nhat cho xe/nguoi/xe dap/xe may phia truoc:

1. Radar LRR thay vat can nam trong corridor.
2. YOLO co box on dinh nam trung voi vat can phia truoc.
3. Tinh TTC va stopping distance.

TTC la thoi gian uoc tinh den va cham:

```text
TTC = distance / closing_speed
```

Neu TTC nho hon nguong theo class thi phanh.

| Class | Nguong TTC phanh |
|---|---:|
| `person` | `2.5 s` |
| `bike_motorbike` | `2.2 s` |
| `car` | `1.8 s` |
| `truck` | `1.5 s` |
| Khac | `2.0 s` |

Ngoai TTC, code con tinh stopping distance:

```text
reaction_distance = speed_mps * AEB_REACTION_TIME_SECONDS
braking_distance = speed_mps^2 / (2 * AEB_MAX_DECELERATION_MPS2)
stopping_distance = (reaction_distance + braking_distance) * AEB_STOPPING_SAFETY_MARGIN
```

Thong so:

```text
AEB_REACTION_TIME_SECONDS = 0.35
AEB_MAX_DECELERATION_MPS2 = 8.0
AEB_STOPPING_SAFETY_MARGIN = 1.30
```

Neu vat can o trong stopping distance, he thong cung co the phanh.

Ket qua:

```text
source = LRR+CAM
reason = lrr_ttc_camera_confirm
target_class = car/person/bike_motorbike/truck
```

### 4.5 Truong hop 5: LRR radar-only cho vat can khong co class YOLO

Day la phan xu ly tuong, cot dien, cay, rao chan, vat can la ma YOLO khong nhan thanh 4 class.

Neu LRR thay cum diem nam trong duong di nhung camera khong confirm, code khong bo qua ngay. No chay logic radar-only:

```text
source = LRR_ONLY
reason = radar_only_obstacle_distance
hoac
reason = radar_only_obstacle_ttc
```

Dieu kien radar-only:

```text
AEB_RADAR_ONLY_ENABLED = True
AEB_RADAR_ONLY_CONFIRM_FRAMES = 2
AEB_RADAR_ONLY_MIN_BRAKE_DISTANCE_M = 5.0
AEB_RADAR_ONLY_MAX_DISTANCE_M = 80.0
AEB_RADAR_ONLY_TTC_BRAKE_SECONDS = 1.20
AEB_RADAR_ONLY_EXTRA_MARGIN_M = 2.0
```

Y nghia:

- Radar-only can it nhat 2 frame lien tiep de giam phanh ao.
- Chi phanh khi vat can trong vung distance/TTC hop ly.
- Dung cho vat can khong thuoc 4 class YOLO.

Nguong khoa cung radar-only:

```text
AEB_RADAR_ONLY_HARD_LOCK_TTC_SECONDS = 0.70
AEB_RADAR_ONLY_HARD_LOCK_DISTANCE_M = 3.5
```

### 4.6 Truong hop 6: camera-only fallback

Neu YOLO thay object lon, nam giua vung nguy hiem, nhung radar chua confirm, code co camera-only fallback.

Dieu kien co ban:

- Box nam trong danger zone ngang anh.
- Box du lon theo ti le chieu cao.
- Track on dinh.

Thong so:

```text
DANGER_ZONE_X_MIN = 0.30
DANGER_ZONE_X_MAX = 0.70
CAMERA_ONLY_HEIGHT_RATIO = 180 / 640
```

Ket qua:

```text
source = CAM_ONLY
reason = camera_only_large_box
```

Camera-only la lop du phong. Logic chinh van la radar hoac radar + camera.

## 5. Cac trang thai AEB

FusionResult co cac truong quan trong:

| Truong | Y nghia |
|---|---|
| `should_warn` | Co can canh bao nguy co |
| `should_brake` | Co yeu cau phanh |
| `brake_force` | Luc phanh, hien tai chu yeu la `1.0` khi nguy hiem |
| `ttc` | Time-to-collision |
| `source` | Nguon kich hoat: `LRR+CAM`, `LRR_ONLY`, `SRR-L`, `SRR-R`, `CAM_ONLY`, `COLLISION` |
| `reason` | Ly do chi tiet |
| `target_class` | Class YOLO neu co |
| `depth` | Khoang cach vat can |
| `closing_speed` | Toc do tiep can |
| `stopping_distance` | Khoang cach can de dung |
| `critical` | Nguy hiem can khoa phanh/manh |

Trang thai hien thi tren HUD:

| Trang thai | Y nghia |
|---|---|
| `AEB OFF` | Nguoi dung tat AEB bang phim `B` |
| `AEB MONITORING` | Dang giam sat, chua phanh |
| `AEB WARNING` | Co rui ro, canh bao |
| `AEB BRAKING` | Dang phanh |
| `AEB CRITICAL - HARD BRAKE` | Nguy hiem gan, phanh cuc dai |
| `AEB LOCKED - RELEASE W` | Da khoa phanh, nguoi lai phai nha ga `W` de xe thoat khoa |

## 6. Co che phanh va khoa ga

Phan apply len xe nam trong `VehicleController.apply_aeb_brake()`.

Khi AEB yeu cau phanh:

```text
throttle = 0.0
brake = brake_force
gear = 1
manual_gear_shift = False
```

Neu la hard lock:

```text
hand_brake = True
target_velocity = 0
target_angular_velocity = 0
```

Thong so:

```text
AEB_HARD_LOCK_USE_HANDBRAKE = True
```

### 6.1 Tai sao bam W van khong duoc thang AEB

Trong `main.py`, neu AEB dang critical hoac dang trong thoi gian lock, chuong trinh chan ga nguoi lai:

```text
throttle_cmd = 0
brake_cmd = AEB brake
```

Muc dich la tranh tinh huong nguoi lai giu `W` lam xe tiep tuc lao vao vat can.

Sau hard brake, neu cau hinh bat:

```text
AEB_REQUIRE_ACCEL_RELEASE_AFTER_CRITICAL = True
```

Thi xe vao trang thai:

```text
AEB LOCKED - RELEASE W
```

Nguoi lai can nha phim `W` / nha ga. Khi xe da gan dung yen hoac nguoi lai nha ga, AEB moi tra quyen ga lai.

Nguong tu dong nha khoa khi xe gan dung:

```text
AEB_ACCEL_BLOCK_RELEASE_SPEED_KMH = 1.0
```

### 6.2 Khi nao hard lock duoc kich hoat

Hard lock thuong xay ra khi:

- SRR thay vat can rat gan.
- LRR-only thay vat can gan hon nguong hard distance.
- LRR+CAM co TTC rat nho.
- Collision sensor da bao va cham.

Nguong LRR hard lock:

```text
AEB_LRR_HARD_LOCK_TTC_SECONDS = 0.80
AEB_LRR_HARD_LOCK_DISTANCE_M = 4.0
```

Thoi gian giu hard brake toi thieu:

```text
AEB_CRITICAL_BRAKE_HOLD_SECONDS = 1.00
```

## 7. Vi sao co luc AEB phanh ao

Phanh ao co the den tu cac nguon sau:

1. Radar nhin thay vat can khong nam trong camera, vi radar co FOV khac camera.
2. Diem radar tu mat duong/tuong/cot nam trong corridor.
3. YOLO nham vat the thanh `person`/`car`.
4. Xe di gan via he, cot den, chau cay; SRR thay vat can ben hong va cho rang xe dang danh lai vao do.
5. Toc do thap nhung radar thay closing speed khong on dinh.

Code da co cac co che giam nhieu:

- Loc cluster radar, khong dung 1 diem don le.
- Loc cao do radar.
- Loc lateral corridor theo duong di xe.
- Radar-only can confirm nhieu frame.
- Tracker giu box on dinh, tranh box nhay bat/tat.
- Loc person false positive va rao chan cong trinh.
- Toc do nho hon `5 km/h` thi khong kich hoat AEB moi.

Neu can giam phanh ao nua, cac nguong thuong dieu chinh la:

| Muc tieu | Tham so nen xem |
|---|---|
| Radar-only phanh qua som | Tang `AEB_RADAR_ONLY_MIN_BRAKE_DISTANCE_M` can than hoac giam `AEB_RADAR_ONLY_TTC_BRAKE_SECONDS` |
| Phanh ao do 1-2 diem radar | Tang `RADAR_CLUSTER_MIN_POINTS`, `RADAR_ONLY_CONFIRM_FRAMES` |
| SRR phanh khi di gan le duong | Tang `AEB_SRR_SIDE_STEER_THRESHOLD`, giam `AEB_SRR_SIDE_LATERAL_MAX_M` |
| YOLO nham nguoi | Tang nguong confidence/lap them filter trong `perception.py` |
| LRR phanh vat can ngoai lan | Giam `RADAR_LRR_CORRIDOR_HALF_WIDTH` hoac `RADAR_LRR_CORRIDOR_GROWTH` |

## 8. Lane keeping hoat dong nhu the nao

Lane keeping gom hai phan:

1. Nhan dien lan duong bang segmentation.
2. Tinh steer assist dua xe ve giua lan.

### 8.1 Lane segmentation runtime

`LaneOverlayRuntime` chay lane model bat dong bo de khong lam tre vong lap chinh.

Luong xu ly:

```text
Camera frame
   -> LatestFrameWorker
   -> LaneSegmentationInferencer
   -> raw lane mask
   -> filter_lane_prediction_geometry
   -> _stabilize_prediction
   -> overlay + lane_prediction
```

Model lane:

```text
aeb/weights/lane_unet_best.onnx
```

Ket qua lane co the gom:

- `ego_lane_mask`: mask lan xe dang di.
- `left_lane_mask`: vach/trai.
- `right_lane_mask`: vach/phai.
- `lane_confidence`: do tin cay.
- `lane_center_x`: tam lan uoc tinh.
- `lane_pixels`: so pixel lane hop le.

### 8.2 Loc hinh hoc lane

Trong `lane_runtime.py`, mask lane khong duoc dung thang. No qua cac loc:

1. Bo phan tren anh theo ROI vi lane can quan tam nam phia duoi.
2. Bo cac hang mask qua rong.
3. Giu component lon nam gan day anh va gan trung tam xe.
4. Morphological close de noi net dut.
5. Tinh lai so pixel va confidence.

Muc dich:

- Giam nham bong cay, via he, vach toa nha.
- Giu lan xe ego thay vi bat nham lan doi dien.
- Giam hien tuong mask nhap nhay.

Thong so dang chu y:

```text
LANE_ROI_TOP_RATIO
LANE_KEEP_BOTTOM_COMPONENT
LANE_BOTTOM_CENTER_HALF_WIDTH_RATIO
LANE_FILTER_CLOSE_KERNEL
```

### 8.3 On dinh lane theo thoi gian

Sau khi co mask moi, code so sanh voi mask truoc:

- Neu lane qua yeu, dung lai lane truoc trong gioi han.
- Neu tam lane nhay qua xa, coi la outlier.
- Neu lane mat tam thoi, giu ket qua cu mot vai frame.
- Neu lane on dinh lai, cap nhat history.

Thong so quan trong:

```text
LANE_STABILIZATION_MAX_CENTER_JUMP_RATIO
```

Dieu nay giup xe khong giat lai khi lane model nham trong 1-2 frame.

## 9. Thuat toan LKAS tinh goc lai

Code chinh nam trong `LaneKeepingAssist.update()`.

LKAS chi chay neu:

```text
LANE_KEEPING_ENABLED = True
toc do >= LANE_KEEPING_MIN_SPEED_KMH
co ego_lane_mask
lane_confidence >= LANE_KEEPING_MIN_CONFIDENCE
pixel_count >= LANE_KEEPING_MIN_PIXELS
```

Gia tri hien tai:

```text
LANE_KEEPING_MIN_SPEED_KMH = 15.0
LANE_KEEPING_MIN_CONFIDENCE = 0.05
LANE_KEEPING_MIN_PIXELS = 120
```

Neu khong du dieu kien, LKAS khong danh lai va tra ve ly do:

| Ly do | Y nghia |
|---|---|
| `disabled` | Nguoi dung tat LKAS bang phim `L` |
| `below_speed` | Toc do duoi nguong |
| `no_lane` | Khong co ket qua lane |
| `no_mask` | Khong co ego lane mask |
| `weak_confidence` | Lane confidence qua thap |
| `weak_lane` | Qua it pixel lane |

### 9.1 Uoc tinh tam lan

LKAS khong lay mot dong duy nhat. No lay nhieu vung ngang quanh diem nhin phia truoc:

```text
lookahead_y_ratio = 0.68
roi_half_height_ratio = 0.055
```

Nghia la no nhin khoang 68% chieu cao anh, quanh vung gan mui xe nhung van du xa de du bao huong lan.

Tai moi vung, code lay vi tri pixel lane, tinh center. Sau do:

- Uu tien cac band dang tin hon bang weight.
- Dung weighted average/median.
- Neu khong co du lieu, fallback ve phan thap cua anh.

### 9.2 Tinh sai so lech tam

Tam muc tieu trong anh:

```text
target_x = image_width * LANE_KEEPING_TARGET_X_RATIO
```

Gia tri hien tai:

```text
LANE_KEEPING_TARGET_X_RATIO = 0.50
```

Sai so:

```text
error_px = lane_center_x - target_x
normalized_error = error_px / image_width
```

Neu sai so nho hon deadband:

```text
LANE_KEEPING_DEADBAND_PX = 8
```

Thi LKAS coi nhu xe dang o giua lan va khong danh lai manh.

### 9.3 Bo dieu khien lai

Steer raw duoc tinh gan theo P + D:

```text
raw_steer = kp * normalized_error + derivative_term
```

Thong so:

```text
LANE_KEEPING_KP = 0.58
LANE_KEEPING_MAX_STEER = 0.35
LANE_KEEPING_SMOOTHING = 0.48
```

Sau do code:

- Clamp steer trong gioi han `max_steer`.
- Gioi han buoc nhay center de tranh giat.
- Lam muot steer qua frame.
- Neu loi lon thi giam smoothing de xe phan ung nhanh hon.

Ket qua tra ve:

```text
active = true/false
steer = gia tri ho tro lai
lane_center_x = tam lan
target_x = tam xe mong muon
lookahead_y = dong anh dung de nhin lan
error_px = sai so pixel
pixel_count = so pixel lane
```

### 9.4 LKAS ket hop voi nguoi lai

Phim:

```text
L: bat/tat LKAS
H: chuyen manual/auto
W/A/S/D: dieu khien xe
B: bat/tat AEB
SPACE: handbrake
```

Trong manual mode, LKAS co the cong them steer assist vao lenh nguoi lai theo he so:

```text
LANE_KEEPING_MANUAL_ASSIST_SCALE = 0.65
```

Trong cau hinh hien tai:

```text
LANE_KEEPING_DRIVER_STEER_SCALE = 0.0
```

Nghia la khi LKAS active, steer cua LKAS co quyen dieu khien ro hon. Neu muon nguoi lai co nhieu quyen hon, tang tham so driver steer scale.

## 10. Man hinh hien thi

Ung dung hien thi:

1. Camera third-person / goc nhin ben ngoai.
2. Camera truoc co box YOLO va overlay lane.
3. Panel radar/AEB o ben duoi.

Panel radar/AEB hien:

- Trang thai AEB.
- Brake hien tai.
- Nguon kich hoat (`SRC`).
- Khoang cach, TTC.
- So diem radar LRR/SRR.
- Mau diem radar:
  - Do: object dang closing/nguy hiem.
  - Vang: static.
  - Xanh: receding.

HUD tren camera hien:

- Toc do.
- FPS.
- Model YOLO.
- Backend ONNX.
- Trang thai lane.
- Trang thai LKAS.
- Che do dieu khien.
- Toggle AEB/LKAS.
- Lenh throttle/brake/steer hien tai.

## 11. Cach doc ket qua test AEB

File ket qua thuong nam trong:

```text
aeb/test_results/aeb_srunner_results.csv
aeb/test_results/c2c_matrix_results.csv
```

Cot quan trong:

| Cot | Y nghia |
|---|---|
| `scenario` | Ten test |
| `result` | `PASS` hoac `FAIL` |
| `collision` | Co va cham hay khong |
| `aeb_triggered` | AEB co tung kich hoat khong |
| `min_distance_m` | Khoang cach nho nhat toi target |
| `brake_start_distance_m` | Khoang cach luc bat dau phanh |
| `brake_start_speed_kmh` | Toc do luc bat dau phanh |
| `final_speed_kmh` | Toc do cuoi |
| `source` | Nguon kich hoat AEB |
| `reason` | Ly do kich hoat |
| `target_class` | Class neu do YOLO confirm |
| `ttc_s` | TTC luc kich hoat |
| `notes` | Ghi chu loi/pass |

Vi du:

```text
PASS,false,true,0.887,6.686,26.55,0.00,26.56,LRR_ONLY,radar_only_obstacle_distance
```

Y nghia:

- Khong va cham.
- AEB da kich hoat.
- Khoang cach gan nhat con lai la `0.887 m`.
- Bat dau phanh luc cach target `6.686 m`.
- Toc do luc phanh `26.55 km/h`.
- Nguon kich hoat la radar-only.

Neu ket qua:

```text
FAIL,true,true,0.000,0.000,...,COLLISION,collision_emergency
```

Nghia la AEB chi kich hoat sau khi collision sensor bao va cham. Day la fail that su, can tune detection/phanh som hon.

## 12. ScenarioRunner va OpenSCENARIO

Huong test hien tai co hai nhom:

### 12.1 Test nhanh mot scenario

Chay client AEB gan vao ego cua ScenarioRunner:

```powershell
python aeb\tools\aeb_srunner_client.py --scenario C2C_FollowLeadingVehicle --target-speed-kmh 35
```

Neu chay co YOLO:

```powershell
python aeb\tools\aeb_srunner_client.py --scenario C2C_FollowLeadingVehicle --target-speed-kmh 35
```

Neu muon radar-only de test vat can khong co class:

```powershell
python aeb\tools\aeb_srunner_client.py --scenario C2C_FollowLeadingVehicle --target-speed-kmh 35 --disable-yolo
```

### 12.2 Test ma tran C2C

Script:

```powershell
powershell -ExecutionPolicy Bypass -File .\aeb\tools\run_c2c_matrix.ps1 -Family CCRs -Speed 30 -Overlap 100
```

Y nghia:

- `Family CCRs`: xe target dung yen.
- `Speed 30`: ego chay 30 km/h.
- `Overlap 100`: can giua hoan toan.

Co the mo rong theo:

```text
CCRs: target dung yen, ego 10-80 km/h
CCRm: target chay 20 km/h, ego nhanh hon
CCRb: target phanh gap khi hai xe dang chay
Overlap: 50%, 75%, 100%
```

Luu y quan trong: neu script/test chay `--disable-yolo` thi ket qua source thuong la `LRR_ONLY`. Luc do dang test logic radar-only, khong phai fusion radar + camera. Neu muon dung day du logic phanh cua app, can chay co YOLO/model.

## 13. Diem manh hien tai

He thong hien tai co cac diem manh:

- Khong phu thuoc hoan toan vao YOLO, vi radar-only co the phanh voi tuong/cot/vat can la.
- Co SRR hai ben de xu ly danh lai vao vat can gan.
- Co khoa ga khi AEB critical, tranh nguoi lai giu `W` lam xe tiep tuc dam.
- Co tracker object de giam nhap nhay detection.
- Co loc lane va on dinh lane theo thoi gian, giup LKAS bot giat.
- Co file CSV de danh gia pass/fail thay vi chi nhin bang mat.

## 14. Gioi han hien tai

Nhung diem can biet khi danh gia:

1. YOLO chi co 4 class, nen vat can khac phu thuoc vao radar.
2. Radar CARLA co nhieu diem nhieu o gan tuong/cot/via he, nen can tune corridor va cluster.
3. Camera-only fallback co the nham neu YOLO nhan sai.
4. LKAS dua vao segmentation; neu lane bi che, bong nang, vach mo, ket qua co the yeu.
5. ScenarioRunner test dung OpenSCENARIO can dam bao ego vehicle con ton tai du lau de client AEB attach vao.
6. Ket qua PASS/FAIL theo CSV la theo tieu chi code hien tai, khong phai chung nhan Euro NCAP chinh thuc.

## 15. Huong tune nhanh

### Muon AEB phanh muon hon

Dieu chinh can than:

```text
AEB_RADAR_ONLY_TTC_BRAKE_SECONDS giam
AEB_RADAR_ONLY_HARD_LOCK_DISTANCE_M giam
AEB_LRR_HARD_LOCK_DISTANCE_M giam
RADAR_LRR_CORRIDOR_HALF_WIDTH giam
```

Rui ro: phanh muon qua se dam.

### Muon AEB phanh som hon

```text
AEB_RADAR_ONLY_TTC_BRAKE_SECONDS tang
AEB_RADAR_ONLY_MIN_BRAKE_DISTANCE_M tang
AEB_STOPPING_SAFETY_MARGIN tang
AEB_TTC_THRESHOLDS tang theo class
```

Rui ro: phanh ao/phanh xa hon.

### Muon giam false positive radar

```text
RADAR_CLUSTER_MIN_POINTS tang
AEB_RADAR_ONLY_CONFIRM_FRAMES tang
RADAR_LRR_CORRIDOR_HALF_WIDTH giam
RADAR_MIN_CLOSING tang nhe
```

### Muon LKAS bam lan manh hon

```text
LANE_KEEPING_KP tang
LANE_KEEPING_MAX_STEER tang nhe
LANE_KEEPING_SMOOTHING giam nhe
```

Rui ro: xe de lac/gat lai.

### Muon LKAS muot hon

```text
LANE_KEEPING_SMOOTHING tang
LANE_KEEPING_KP giam nhe
LANE_KEEPING_DEADBAND_PX tang
```

Rui ro: phan ung cham khi vao cua.

## 16. Tom tat ngan gon

AEB hien tai la fusion giua:

```text
YOLO camera + radar LRR + radar SRR trai/phai + collision sensor
```

Trong do:

- YOLO giup xac dinh class vat the.
- LRR giup do khoang cach/TTC phia truoc.
- SRR giup bat vat can gan va vat can khi danh lai vao hai ben.
- Radar-only xu ly vat can khong nam trong 4 class YOLO.
- Hard lock chan ga va co the dung handbrake de tranh nguoi lai giu `W` dam tiep.

Lane keeping hien tai la:

```text
Lane segmentation -> loc mask -> on dinh lane -> tinh tam lan -> P/D steering assist
```

Trong do LKAS chi active khi lane du tin cay va toc do tren nguong, co lam muot va chong giat de xe bam lan on dinh hon.
