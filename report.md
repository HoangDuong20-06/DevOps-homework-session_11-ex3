# Bao cao Bai 3: Toi uu toc do CI bang Gradle Dependency Caching

## 1. Cau hinh

File `.github/workflows/ci.yml` (JDK 21 cho khop toolchain trong `build.gradle`, distribution `temurin`):

```yaml
name: CI
on:
  push:
  pull_request:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Thiet lap moi truong Java JDK 21
        uses: actions/setup-java@v5
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
      - name: Cap quyen va chay test voi Gradle
        run: |
          chmod +x gradlew
          ./gradlew test
```

Diem quan trong: `cache: 'gradle'` trong step `actions/setup-java@v5`.
No tu dong cache `~/.gradle/caches` + `~/.gradle/wrapper` theo hash cua `*.gradle*` va `gradle-wrapper.properties`.

> Ghi chu: de bai goi y JDK 17, nhung repo nay dung toolchain Java 21 (`build.gradle`)
> nen dung JDK 21 de CI pass. Chi can doi `java-version: '17'` neu thay bat buoc.

## 2. Ket qua 2 lan push lien tiep

Repo: https://github.com/HoangDuong20-06/DevOps-homework-session_11-ex3

| Lan push | Commit | Run | Trang thai | Tong thoi gian run | Job `build` |
|---|---|---|---|---|---|
| Lan 1 (chua cache - MISS) | `cefb094` - `ci: them workflow voi gradle cache` | [#1](https://github.com/HoangDuong20-06/DevOps-homework-session_11-ex3/actions/runs/38141155394) | success | ~50s (`12:55:53Z` -> `12:56:43Z`) | ~47s (`12:55:55Z` -> `12:56:42Z`) |
| Lan 2 (da cache - HIT) | `92daed3` - `ci: push lan 2 de kiem tra gradle cache` | [#2](https://github.com/HoangDuong20-06/DevOps-homework-session_11-ex3/actions/runs/38141270546) | success | ~22s (`12:57:41Z` -> `12:58:03Z`) | ~19s (`12:57:43Z` -> `12:58:02Z`) |

Chi tiet tung step (lay tu GitHub Actions API `.../actions/runs/<id>/jobs`):

| Step | Lan 1 (MISS) | Lan 2 (HIT) | Chenh lech |
|---|---|---|---|
| `Thiet lap moi truong Java JDK 21` (setup-java) | 1s (`12:55:58` -> `12:55:59`) | 3s (`12:57:45` -> `12:57:48`, gom restore cache) | +2s (do giai nen cache, nhung tiet kiem download) |
| `Cap quyen va chay test voi Gradle` (`./gradlew test`) | 36s (`12:55:59` -> `12:56:35`) | 12s (`12:57:48` -> `12:58:00`) | **-24s (~-67%)** |
| `Post Thiet lap moi truong Java JDK 21` (luu/khong luu cache) | 5s (`12:56:35` -> `12:56:40`, luu cache moi) | 1s (`12:58:00` -> `12:58:01`, khong can luu lai) | **-4s** |
| Tong Job | ~47s | ~19s | **-28s (~-60%)** |
| Tong Run | ~50s | ~22s | **-28s (~-56%)** |

## 3. Kiem tra log cache

- Lan 1: step `Thiet lap moi truong Java JDK 21` bao `Cache not found for input keys: gradle-...`
  ( + step `Post ...` mat 5s de upload cache).
- Lan 2: step `Thiet lap moi truong Java JDK 21` xuat hien dong:
  ```
  Cache restored successfully from key: gradle-...
  ```
  Cach xem: GitHub -> tab `Actions` -> mo run #2 -> Job `build` ->
  mo step `Thiet lap moi truong Java JDK 21`.

## 4. Ket luan

- Bat `cache: 'gradle'` giup Job giam tu ~47s xuong ~19s (~60%), chu yeu nho
  step `./gradlew test` khong phai tai lai dependencies tu Maven Central (36s -> 12s).
- Push lan 2 khong can upload cache lai (Post step 5s -> 1s).
- Dat yeu cau: co `cache: 'gradle'`, log lan 2 co `Cache restored successfully`,
  thoi gian chay giam dang ke.
