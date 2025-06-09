# 🕵️‍♂️ Bug Bounty Unified Recon & Automation Guide (2025)

## الفهرس
- [المرحلة 0: الإعداد الذكي](#المرحلة-0-الإعداد-الذكي)
- [المرحلة 1: الاستطلاع المتقدم](#المرحلة-1-الاستطلاع-المتقدم)
- [المرحلة 2: الزحف والتحليل الديناميكي](#المرحلة-2-الزحف-والتحليل-الديناميكي)
- [المرحلة 3: الفحص الآلي المتقدم](#المرحلة-3-الفحص-الآلي-المتقدم)
- [المرحلة 4: الاختبار اليدوي المتعمق](#المرحلة-4-الاختبار-اليدوي-المتعمق)
- [المرحلة 5: تحليل النتائج والإبلاغ](#المرحلة-5-تحليل-النتائج-والإبلاغ)
- [روابط وأدلة سريعة](#روابط-وأدلة-سريعة)
- [سكربت الأتمتة الرئيسي](#سكربت-الأتمتة-الرئيسي)

---

## المرحلة 0: الإعداد الذكي (Pre-Engagement)
**الهدف:** تجهيز بيئة عمل منظمة وقابلة للتخصيص.

```bash
# إعداد متغيرات بيئية ديناميكية
echo "export TARGET=target.com" > .env
echo "export THREADS=50" >> .env
echo "export OUTPUT_DIR=$(date +'%Y-%m-%d_scan')" >> .env
source .env
mkdir -p $OUTPUT_DIR/{subdomains,urls,scans,exploits,reports,workflows,logs,apis}
cd $OUTPUT_DIR
git init
echo "# Recon Project Structure" > README.md
```
**ليش؟**
- تخصيص سهل للأهداف وعدد الـ threads.
- تنظيم النتائج في مجلدات بتاريخ التنفيذ.

---

## المرحلة 1: الاستطلاع المتقدم (Advanced Recon)
**الهدف:** بناء قائمة شاملة بالنطاقات الفرعية والمعلومات التقنية.

### جمع النطاقات الفرعية (Subdomain Enumeration)
```bash
subfinder -d $TARGET -o subdomains/subfinder.txt
amass enum -d $TARGET -passive -o subdomains/amass.txt
assetfinder --subs-only $TARGET | sort -u > subdomains/assetfinder.txt
curl -s "https://crt.sh/?q=%.$TARGET&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u > subdomains/crtsh.txt
cat subdomains/*.txt | sort -u > subdomains/all_unique.txt
```

### فحص النطاقات الحية (بدون httpx)
```bash
cat subdomains/all_unique.txt | httprobe > subdomains/alive.txt
```
**ليش؟**
- httprobe: فحص النطاقات الحية بسرعة وبدون تعقيد.

### كشف الـ CDN (اختياري)
```bash
if command -v cdnhunter &> /dev/null; then
  cat subdomains/alive.txt | cdnhunter -o subdomains/cdn_ips.txt
fi
```
**ليش؟**
- معرفة إذا كان النطاق خلف CDN لتجنب الحظر.

### استخدام shuffledns وmassdns (اختياري)
```bash
shuffledns -d $TARGET -list subdomains/all_unique.txt -r resolvers.txt -mode resolve -o subdomains/shuffledns_resolved.txt
cat subdomains/shuffledns_resolved.txt >> subdomains/alive.txt

massdns -r resolvers.txt -t A -o S subdomains/all_unique.txt > subdomains/massdns.txt
cat subdomains/massdns.txt | awk '{print $1}' | sed 's/\.$//' >> subdomains/alive.txt
```

### توحيد النتائج النهائية للنطاقات الفرعية
```bash
cat subdomains/alive.txt | sort -u > subdomains/final.txt
```

---

## المرحلة 2: الزحف والتحليل الديناميكي (Dynamic Crawling & Analysis)
**الهدف:** اكتشاف أكبر عدد من الروابط ونقاط النهاية.

### الزحف العميق (Deep Crawling)
```bash
katana -u https://$TARGET -jc -kf -o urls/katana.txt
gospider -s https://$TARGET -o urls/gospider -t $THREADS -c 10
cat urls/katana.txt urls/gospider/*.txt | sort -u > urls/all_urls_raw.txt
```

### تنظيف وتوحيد الروابط (باستخدام unfurl أو uro)
```bash
if command -v unfurl &> /dev/null; then
  cat urls/all_urls_raw.txt | unfurl -u format %s://%d%p > urls/all_urls_clean.txt
else
  cat urls/all_urls_raw.txt | uro > urls/all_urls_clean.txt
fi
cat urls/all_urls_clean.txt | sort -u > urls/final.txt
```
**ليش؟**
- unfurl يحافظ على الباراميترات المهمة في الروابط.

### فحص البورتات السريع (اختياري)
```bash
if command -v rustscan &> /dev/null; then
  rustscan -a $TARGET --ulimit 5000 -p 1-65535 --scan-order "Random" -g scans/ports.txt
fi
```
**ليش؟**
- RustScan أسرع بكثير من nmap.

### استخراج نقاط النهاية وأسرار JS
```bash
for js in $(grep -iE "\.js$" urls/final.txt); do linkfinder.py -i $js -o cli | tee -a urls/js_endpoints.txt; done
for js in $(grep -iE "\.js$" urls/final.txt); do curl -s $js | grep -E "api_key|password|token" | sed "s|^|$js : |" >> urls/js_secrets.txt; done
```

### فحص إحصائي باستخدام gf
```bash
gf sqli urls/final.txt | sort -u > urls/sqli_endpoints.txt
gf xss urls/final.txt | sort -u > urls/xss_endpoints.txt
cat urls/xss_endpoints.txt | httprobe > urls/xss_alive.txt
```

---

## المرحلة 3: الفحص الآلي المتقدم (Advanced Automated Scanning)
**الهدف:** استخدام أدوات الفحص الآلي لاكتشاف الثغرات.

### فحص Nuclei (مع Smart Templates وفلترة النتائج)
```bash
nuclei -update-templates
nuclei -l urls/final.txt -nt -as -severity medium,high -o scans/nuclei_smart.txt
grep -v "404" scans/nuclei_smart.txt | grep -v "Not Vulnerable" > scans/nuclei_filtered.txt
```
**ليش؟**
- Smart Templates تقلل الضجيج وتزيد الدقة.

### فحص GraphQL وAPI وServerless (اختياري)
```bash
grep -i "graphql" urls/final.txt | httprobe > apis/graphql_endpoints.txt
for url in $(cat apis/graphql_endpoints.txt); do graphqlmap -u $url --method POST --dump | tee -a scans/graphql/graphqlmap_results.txt; done

if command -v wafbuster &> /dev/null; then
  wafbuster -t $TARGET -m api -o scans/wafbuster_api.txt
fi
```
**ليش؟**
- WAFBuster يغطي ثغرات API الحديثة.

---

## المرحلة 4: الاختبار اليدوي المتعمق (In-Depth Manual Testing)
**الهدف:** التحقق يدويًا من الثغرات واكتشاف منطق الأعمال.

### أدوات مساعدة:
- **Teler** لمراقبة الهجمات الحية:  
  `docker run -d teler -p 8080:8080`
- **AutoRepeater** (Burp Suite) لأتمتة هجمات التكرار.

### اختبار SSRF وXXE
```bash
cat urls/ssrf_potential.txt | while read url; do ffuf -u "$url=FUZZ" -w ~/payloads/ssrf.txt -o exploits/ssrf_results.txt; done
```

### اختبار XSS وSQLi يدويًا
- استخدم Burp Suite مع ملفات gf الناتجة.

### اختبار منطق الأعمال وتجاوز Rate Limit
```bash
curl -X POST "https://shop.example.com/cart" -d "item=123&price=0.01" -H "Cookie: session=YOUR_SESSION" -o exploits/price_manipulation.txt
```

---

## المرحلة 5: تحليل النتائج والإبلاغ (Analysis & Reporting)
**الهدف:** دمج النتائج وتحليلها وإعداد تقرير نهائي.

### 5.1 دمج النتائج النهائية
```bash
cat scans/nuclei_filtered.txt scans/graphql/graphqlmap_results.txt scans/wafbuster_api.txt > scans/all_findings_raw.txt
```

### 5.2 تحليل أولي باستخدام الذكاء الاصطناعي مع مراجعة بشرية
**الهدف:** تسريع فرز الثغرات وتحديد أولويات الإصلاح، مع مراجعة بشرية نهائية.

#### الخطوات:
1. **تحويل النتائج إلى JSON Structured (أو CSV):**
   ```bash
   awk '{print $1, $2, $3}' scans/all_findings_raw.txt > scans/all_findings_summary.txt
   cat scans/all_findings_summary.txt | jq -R -s -c 'split("\n")[:-1] | map(split(" ")) | map({"severity":.[0], "url":.[1], "desc":.[2]})' > scans/findings.json
   ```
2. **تحليل النتائج بالذكاء الاصطناعي (OpenAI API):**
   ```bash
   openai api completions.create -m "gpt-4o-mini" \
     -p "Analyze this list of security findings, prioritizeها حسب الأهمية والأثر المحتمل، واقترح خطوات إصلاح لكل منها. List in JSON with fields: {url, type, severity, recommended_fix}." \
     -i scans/findings.json \
     -o scans/ai_prioritized.json
   ```
3. **مراجعة بشرية نهائية:**
   - راجع يدويًا كل ثغرة (خصوصًا الحساسة منها) للتأكد من دقة التحليل والتوصيات.
   - أضف ملاحظاتك في عمود إضافي (review_notes) في الملف:
   ```bash
   # مثال إضافة ملاحظات يدوية
   # يمكنك استخدام jq أو أي محرر نصوص لإضافة review_notes لكل ثغرة
   cp scans/ai_prioritized.json scans/ai_prioritized_reviewed.json
   # ثم أضف الملاحظات يدويًا أو عبر سكربت بسيط
   ```

**ليش؟**
- الذكاء الاصطناعي يسرّع فرز الثغرات وتحديد الأولويات، لكن المراجعة البشرية ضرورية لتجنب الأخطاء أو التوصيات غير المنطقية.

### 5.3 توليد التقارير النهائية
```bash
# توليد تقارير SARIF (تكامل مع GitHub/GitLab)
nuclei -export sarif -o reports/report.sarif

# توليد تقرير DefectDojo (إذا متوفر)
if command -v nuclei &> /dev/null; then
  nuclei -l scans/all_findings_raw.txt -report defectdojo -target $TARGET
fi
```
**ليش؟**
- SARIF: تكامل مع أنظمة DevSecOps.
- DefectDojo: إدارة الثغرات بشكل احترافي.

---

## سكربت الأتمتة الرئيسي (run_recon.sh)
```bash
#!/bin/bash
source .env

RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'

# التحقق من وجود الأدوات الأساسية
for tool in subfinder amass assetfinder httprobe nuclei katana gospider shuffledns massdns unfurl uro jq; do
  if ! command -v $tool &> /dev/null; then
    echo -e "${RED}$tool not found! Please install it before running the script.${NC}" | tee -a logs/install_errors.log
    exit 1
  fi
done

# التحقق من وجود resolvers.txt
if [ ! -f resolvers.txt ]; then
  echo -e "${YELLOW}Downloading resolvers.txt...${NC}"
  curl -s https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt -o resolvers.txt
fi

echo -e "${GREEN}[1] Subdomain Enumeration...${NC}"
subfinder -d $TARGET -o subdomains/subfinder.txt
amass enum -d $TARGET -passive -o subdomains/amass.txt
assetfinder --subs-only $TARGET | sort -u > subdomains/assetfinder.txt
curl -s "https://crt.sh/?q=%.$TARGET&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u > subdomains/crtsh.txt
cat subdomains/*.txt | sort -u > subdomains/all_unique.txt
cat subdomains/all_unique.txt | httprobe > subdomains/alive.txt
shuffledns -d $TARGET -list subdomains/all_unique.txt -r resolvers.txt -mode resolve -o subdomains/shuffledns_resolved.txt
cat subdomains/shuffledns_resolved.txt >> subdomains/alive.txt
cat subdomains/alive.txt | sort -u > subdomains/final.txt

echo -e "${GREEN}[2] Crawling URLs...${NC}"
katana -u https://$TARGET -jc -kf -o urls/katana.txt
gospider -s https://$TARGET -o urls/gospider -t $THREADS -c 10
cat urls/katana.txt urls/gospider/*.txt | sort -u > urls/all_urls_raw.txt
if command -v unfurl &> /dev/null; then
  cat urls/all_urls_raw.txt | unfurl -u format %s://%d%p > urls/all_urls_clean.txt
else
  cat urls/all_urls_raw.txt | uro > urls/all_urls_clean.txt
fi
cat urls/all_urls_clean.txt | sort -u > urls/final.txt

echo -e "${GREEN}[3] Automated Scanning...${NC}"
nuclei -l urls/final.txt -nt -as -severity medium,high -o scans/nuclei_smart.txt
grep -v "404" scans/nuclei_smart.txt | grep -v "Not Vulnerable" > scans/nuclei_filtered.txt

echo -e "${GREEN}[4] Filtering & Reporting...${NC}"
cat scans/nuclei_filtered.txt > scans/all_findings_raw.txt
awk '{print $1, $2, $3}' scans/all_findings_raw.txt > scans/all_findings_summary.txt

# إشعار عند الانتهاء (اختياري)
if [[ ! -z "$BOT_TOKEN" && ! -z "$CHAT_ID" ]]; then
  curl -s "https://api.telegram.org/bot$BOT_TOKEN/sendMessage?chat_id=$CHAT_ID&text=Recon+Completed+for+$TARGET"
fi

echo -e "${YELLOW}[+] Recon Workflow Finished!${NC}"
```

---

### روابط وأدلة سريعة
- [assetfinder](https://github.com/tomnomnom/assetfinder)
- [bbot](https://github.com/blacklanternsecurity/bbot)
- [amass](https://github.com/OWASP/Amass)
- [subfinder](https://github.com/projectdiscovery/subfinder)
- [waybackurls](https://github.com/tomnomnom/waybackurls)
- [katana](https://github.com/projectdiscovery/katana)
- [gospider](https://github.com/jaeles-project/gospider)
- [nuclei](https://github.com/projectdiscovery/nuclei)
- [shuffledns](https://github.com/projectdiscovery/shuffledns)
- [uro](https://github.com/s0md3v/uro)
- [httprobe](https://github.com/tomnomnom/httprobe)
- [unfurl](https://github.com/tomnomnom/unfurl)
- [rustscan](https://github.com/RustScan/RustScan)
- [cdnhunter](https://github.com/zeekay/cdnhunter)
- [wafbuster](https://github.com/0xInfection/wafbuster)
- [teler](https://github.com/kitabisa/teler)

---
