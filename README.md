# 성능 개선 보고서
## 용어 설명
1. [First Contentful Paint(FCP)](https://developer.chrome.com/docs/lighthouse/performance/first-contentful-paint?utm_source=lighthouse&utm_medium=lr&hl=ko)  
   콘텐츠가 포함된 첫 페인트는 첫 번째 텍스트 또는 이미지가 표시되는 시간을 나타냅니다.
2. [Largest Contentful Paint(LCP)](https://developer.chrome.com/docs/lighthouse/performance/lighthouse-largest-contentful-paint?utm_source=lighthouse&utm_medium=lr&hl=ko)  
   최대 콘텐츠 렌더링 시간은 최대 텍스트 또는 이미지가 표시되는 시간을 나타냅니다.
3. [Total Blocking Time(TBT)](https://developer.chrome.com/docs/lighthouse/performance/lighthouse-total-blocking-time?utm_source=lighthouse&utm_medium=lr&hl=ko)  
   FCP와 상호작용 시간 사이의 모든 시간의 합을 나타냅니다.
4. [Cumulative Layout Shift(CLS)](https://web.dev/articles/cls?utm_source=lighthouse&utm_medium=lr&hl=ko)  
   레이아웃 변경 횟수는 표시 영역 안에 보이는 요소의 이동을 측정합니다.
## 개선 표
|개선 항목|개선이유|개선 후 향상된 지표|개선 방법|
|---------|---------|-------------------|----------|
|**Compress and resize images**|WebP 및 AVIF와 같은 이미지 형식은 PNG나 JPEG보다 압축률이 높기 때문에 다운로드가 빠르고 데이터 소비량도 적음|리소스 크기 2,433.9 KiB에서, 1,979.8 KiB 만큼 절감|제공된 JPG 이미지를 [squoosh](https://squoosh.app/)에서 Webp 형식으로 변환 및 렌더링 크기에 맞게 리사이즈|
|**Set responsive image source based on screen size**|스크린 크기에 따라서 적절한 크기의 이미지 개제 시 모바일 데이터를 절약하고 로드 시간을 단축 시킬 수 있음|리소스 크기 149.9KiB KiB에서, 88.7 KiB 만큼 절감 |**picture element**를 사용해서 각 브레이크 포인트에 맞는 크기의 이미지를 제공|
|**Set width & height to hero images**|이미지 요소에 명시적인 너비 및 높이를 설정하여 레이아웃 변경 횟수를 줄이고 CLS를 개선할 수 있음|**Cumulative Layout Shift(CLS): 0.016 → 0.011**|히어로 이미지에 명시적인 width & height를 설정|
|**Prevent cumulative layout shift of country banner**|비동기 처리에 의해 DOM요소가 갑자기 생겨나면 **layout shift**가 생겨나고 이는 사용자 경험에 좋지 않음|**CLS: 0.011→ 0.009**|banner에 있던 display: none 속성을 제거하고 height 설정, layout shift를 방지|
|**Image lazy load**|이미지가 화면에 보여질 필요가 없음에도 리소스를 불러오면 로드하는데 시간이 더 걸리고 데이터도 많이 소비|이미지 16개 로드 → 1.047111MB ⇒ **6개 로드 → 452.11100kB**|img element의 lazy 프로퍼티 사용|
|**Self host fonts**|외부 리소스를 불러오는 대신, 폰트 파일을 직접 제공함으로써 웹 페이지 로딩 속도 최적화|리소스 전송 크기 15.6KiB, 전송 시간 200ms 절약|public 폴더안에 font 폴더에 self host font 위치|
|**ttf → woff2**| woff2는 ttf보다 더 나은 압축률 가짐|**LCP 개선**|폰트 포맷을 ttf에서 woff2로 변경|
|**Eliminate render blocking resoures**|리소스가 페이지 첫 페인트를 차단함 이는 FCP, LCP에 영향을 줌|	전송 크기 58.3 KiB, 전송 시간 250 ms 절약|당장 필요하지 않은 스크립트에 defer 속성 명시 & 필요한 리소스는 preload<sub>*1)</sub>|
|**Split heavy js task**|무거운 작업이 있을 때 브라우저에 지연 현상이 발생|JavaScript excution time, TBT 개선|큰 태스크를 쪼개어 실행<sub>*2)</sub>|

## 코드
**1) Eliminate render blocking resoures**


스크립트 속성에 defer를 설정해주면, 페이지가 모두 로드된 후에 해당 외부 스크립트가 실행된다.  
그리고 css 파일은 페이지 페인트에 필수임으로, 빠르게 로딩하도록 preload 속성을 통해 우선순위를 부여할 수 있다.


이것들로 첫 페인트 차단을 방지 할 수 있다.
   


**styles.css**
```
<link rel="preload" href="/css/styles.css" as="style" onload="this.onload=null;this.rel='stylesheet'">
<noscript><link rel="stylesheet" href="/css/styles.css"></noscript>
```


**cookie-consent.js**
```
<script defer type="text/javascript" src="//www.freeprivacypolicy.com/public/cookie-consent/4.1.0/cookie-consent.js" charset="UTF-8"></script>
<script defer type="text/javascript" charset="UTF-8">
    document.addEventListener('DOMContentLoaded', function () {
        cookieconsent.run({"notice_banner_type":"simple","consent_type":"express","palette":"light","language":"en","page_load_consent_levels":["strictly-necessary"],"notice_banner_reject_button_hide":false,"preferences_center_close_button_hide":false,"page_refresh_confirmation_buttons":false,"website_name":"Performance Course"});
    });
</script> 
```

**2) Split heavy js task**


loadProducts()는 페이지 첫 로딩에는 필요 없고 해당 element가 뷰포트에 들어올 때 가져오면 LCP 등에 유리하다. 따라서 스크롤을 내려 해당 element가 뷰포트에 들어올 때 로드하도록 getBoundingClientRect()를 사용한다.


또는 Intersection Observer API를 사용해서 특정 element가 Intersecting 할 때 불러와도 괘찮아 보인다.

```
window.onload = () => {
    let status = 'idle';
    let productSection = document.querySelector('#all-products');
    window.onscroll = () => {
        let position = productSection.getBoundingClientRect().top - (window.scrollY + window.innerHeight);

        if (status == 'idle' && position <= 0) {
            loadProducts();
            performTask()
        }
    }
}

let i = 0;

function performTask() {
  do {
    const temp = Math.sqrt(i) * Math.sqrt(i);
    i++;
  } while (i % 1e6 !== 0 && i < 1e7);  

  if (i < 1e7) {
    setTimeout(performTask);  
  }
}

```

## PageSpeed Insights
![image](https://github.com/user-attachments/assets/61fbaac9-acb3-4e1a-84df-0c78add0bc3c)

