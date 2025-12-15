gemini yg bagus  
import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, collection } from 'firebase/firestore';

// --- FIREBASE CONFIGURATION & GLOBALS ---
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
// Cek apakah config ada, jika tidak gunakan objek kosong agar tidak error
const firebaseConfig = typeof __firebase_config !== 'undefined' && __firebase_config ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// --- TIME UTILITIES ---

// Helper function to convert HH:MM to Date object for today
const getPrayerDate = (prayerTime) => {
    const now = new Date();
    const [hour, minute] = prayerTime.split(':').map(Number);
    return new Date(now.getFullYear(), now.getMonth(), now.getDate(), hour, minute, 0);
};

// Check if the current time is past the prayer time
const isTimePassed = (prayerTime) => {
    return new Date().getTime() > getPrayerDate(prayerTime).getTime() + (60 * 1000);
};

// Check if the current time is within 10 minutes BEFORE the prayer time (Early Completion Window)
const isWithinEarlyWindow = (prayerTime) => {
    const nowMs = new Date().getTime();
    const adzanTimeMs = getPrayerDate(prayerTime).getTime();
    const tenMinutesBeforeAdzanMs = adzanTimeMs - (10 * 60 * 1000); 
    return nowMs < adzanTimeMs && nowMs >= tenMinutesBeforeAdzanMs;
};

// Function to format HH:MM to H:MM AM/PM
const formatTime12h = (time24h) => {
    if (!time24h) return 'XX:XX';
    const [h, m] = time24h.split(':').map(Number);
    const date = new Date(2000, 0, 1, h, m);
    return date.toLocaleTimeString('en-US', { 
        hour: 'numeric', 
        minute: '2-digit', 
        hour12: true 
    }).replace(' AM', ' pagi').replace(' PM', ' sore');
};

// --- QIBLA UTILITIES ---
const calculateQiblaAngle = (userLat, userLon) => {
    const KAABA_LAT = 21.4225;
    const KAABA_LON = 39.8262;
    const latRad = userLat * (Math.PI / 180);
    const kaabaLatRad = KAABA_LAT * (Math.PI / 180);
    const deltaLonRad = (KAABA_LON - userLon) * (Math.PI / 180);

    const y = Math.sin(deltaLonRad) * Math.cos(kaabaLatRad);
    const x = Math.cos(latRad) * Math.sin(kaabaLatRad) -
              Math.sin(latRad) * Math.cos(kaabaLatRad) * Math.cos(deltaLonRad);

    let bearing = Math.atan2(y, x) * (180 / Math.PI);
    if (bearing < 0) bearing += 360;
    return bearing; 
};

// --- HAID/MOON UTILITIES ---
const getHaidStatus = (haidData) => {
    const defaultStatus = { isHaid: false, haidDay: 0, totalDays: 0 };
    if (!haidData || !haidData.startDate || !haidData.endDate) return defaultStatus;

    const today = new Date();
    today.setHours(0, 0, 0, 0);
    const startDate = new Date(haidData.startDate);
    startDate.setHours(0, 0, 0, 0);
    const endDate = new Date(haidData.endDate);
    endDate.setHours(0, 0, 0, 0);

    if (startDate > endDate || isNaN(startDate.getTime()) || isNaN(endDate.getTime())) return defaultStatus;

    if (today.getTime() >= startDate.getTime() && today.getTime() <= endDate.getTime()) {
        const msPerDay = 1000 * 60 * 60 * 24;
        const diffTime = today.getTime() - startDate.getTime();
        const haidDay = Math.floor(diffTime / msPerDay) + 1; 
        const totalDurationMs = endDate.getTime() - startDate.getTime();
        const totalDays = Math.floor(totalDurationMs / msPerDay) + 1;

        return { isHaid: true, haidDay, totalDays };
    }
    return defaultStatus;
};

// --- ICON COMPONENTS ---
const LocationIcon = ({ className }) => (<svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z"/><circle cx="12" cy="10" r="3"/></svg>);
const HomeIcon = ({ className }) => (<svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M3 9l9-7 9 9v11a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2z"/><polyline points="9 22 9 12 15 12 15 22"/></svg>);
const CheckCircleIcon = ({ className }) => (<svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M22 11.08V12a10 10 0 1 1-5.93-9.14"/><polyline points="22 4 12 14.01 9 11.01"/></svg>);
const QuranIcon = ({ className }) => (<svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M14 2v10l-2-2-2 2V2"/><path d="M5 2h14c1.1 0 2 .9 2 2v16c0 1.1-.9 2-2 2H5c-1.1 0-2-.9-2-2V4c0-1.1.9-2 2-2z"/><path d="M12 16h.01"/></svg>);
const MoonIcon = ({ className }) => (<svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 3a6 6 0 0 0 9 9 9 9 0 1 1-9-9Z"/></svg>);
const SparkleIcon = ({ className }) => (<svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 2v2"/><path d="M12 20v2"/><path d="M4.93 4.93l1.41 1.41"/><path d="M17.66 17.66l1.41 1.41"/><path d="M2 12h2"/><path d="M20 12h2"/><path d="M4.93 19.07l1.41-1.41"/><path d="M17.66 6.34l1.41-1.41"/><circle cx="12" cy="12" r="3"/></svg>);
const CompassIcon = ({ className }) => (<svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M12 21a9 9 0 0 0-9-9 9 9 0 0 0 9-9 9 9 0 0 0 9 9 9 9 0 0 0-9 9"/><path d="m14 10-2 2-2-2"/><path d="M12 21V12"/><path d="M12 12h9"/></svg>);
const ChevronRight = ({ className }) => (<svg className={className} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><polyline points="9 18 15 12 9 6"></polyline></svg>);
const ArrowLeft = ({ className, onClick }) => (<svg className={className} onClick={onClick} xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="m12 19-7-7 7-7"/><path d="M19 12H5"/></svg>);


// Thematic Background
const ThematicBackground = () => (
    <svg className="absolute bottom-0 right-0 w-full h-full opacity-100" viewBox="0 0 1000 700" preserveAspectRatio="xMidYMax slice" xmlns="http://www.w3.org/2000/svg" fill="none">
        <defs>
            <linearGradient id="desertGradient" x1="0%" y1="0%" x2="0%" y2="100%">
                <stop offset="0%" style={{stopColor: "#F59E0B", stopOpacity: 1}} />
                <stop offset="100%" style={{stopColor: "#FCD34D", stopOpacity: 1}} />
            </linearGradient>
            <linearGradient id="duneGradient" x1="0%" y1="0%" x2="0%" y2="100%">
                <stop offset="0%" style={{stopColor: "#C2410C", stopOpacity: 1}} />
                <stop offset="100%" style={{stopColor: "#F59E0B", stopOpacity: 1}} />
            </linearGradient>
        </defs>
        <rect x="0" y="0" width="1000" height="700" fill="url(#desertGradient)" />
        <circle cx="150" cy="100" r="70" fill="#FEF3C7" opacity="0.8"/> 
        <path d="M 0 700 V 550 C 150 500, 300 650, 500 600 L 500 700 Z" fill="#92400E" opacity="0.8" /> 
        <path d="M 400 700 V 500 C 600 400, 800 550, 1000 450 L 1000 700 Z" fill="url(#duneGradient)" opacity="1" />
        <g fill="#78350F" opacity="0.9"> 
            <path d="M 600 550 L 610 540 L 620 550 L 610 560 Z" />
            <path d="M 640 540 L 650 530 L 660 540 L 650 550 Z" />
        </g>
    </svg>
);

// --- DATA: Diperbarui dengan struktur 'verses' untuk tampilan per-ayat ---
const SHORT_SURAHS = [
    { 
        id: 1, name: "Al-Fatihah", ayat: 7, arabic: "بِسْمِ ٱللَّهِ ٱلرَّحْمَٰنِ ٱلرَّحِيمِ", translation: "Pembukaan", 
        verses: [
            { ar: "بِسْمِ ٱللَّهِ ٱلرَّحْمَٰنِ ٱلرَّحِيمِ", id: "Dengan menyebut nama Allah Yang Maha Pengasih lagi Maha Penyayang." },
            { ar: "ٱلْحَمْدُ لِلَّهِ رَبِّ ٱلْعَٰلَمِينَ", id: "Segala puji bagi Allah, Tuhan seluruh alam." },
            { ar: "ٱلرَّحْمَٰنِ ٱلرَّحِيمِ", id: "Yang Maha Pengasih, Maha Penyayang." },
            { ar: "مَٰلِكِ يَوْمِ ٱلدِّينِ", id: "Pemilik hari Pembalasan." },
            { ar: "إِيَّاكَ نَعْبُدُ وَإِيَّاكَ نَسْتَعِينُ", id: "Hanya kepada Engkaulah kami menyembah dan hanya kepada Engkaulah kami mohon pertolongan." },
            { ar: "ٱهْدِنَا ٱلصِّرَٰطَ ٱلْمُسْتَقِيمَ", id: "Tunjukilah kami jalan yang lurus," },
            { ar: "صِرَٰطَ ٱلَّذِينَ أَنْعَمْتَ عَلَيْهِمْ غَيْرِ ٱلْمَغْضُوبِ عَلَيْهِمْ وَلَا ٱلضَّآلِّينَ", id: "(yaitu) jalan orang-orang yang telah Engkau beri nikmat, bukan (jalan) mereka yang dimurkai, dan bukan (pula jalan) mereka yang sesat." },
        ]
    },
    { 
        id: 2, name: "An-Nas", ayat: 6, arabic: "قُلْ أَعُوذُ بِرَبِّ ٱلنَّاسِ", translation: "Manusia", 
        verses: [
            { ar: "قُلْ أَعُوذُ بِرَبِّ ٱلنَّاسِ", id: "Katakanlah (Muhammad), 'Aku berlindung kepada Tuhannya manusia," },
            { ar: "مَلِكِ ٱلنَّاسِ", id: "Raja manusia," },
            { ar: "إِلَٰهِ ٱلنَّاسِ", id: "sembahan manusia," },
            { ar: "مِن شَرِّ ٱلْوَسْوَاسِ ٱلْخَنَّاسِ", id: "dari kejahatan (bisikan) setan yang bersembunyi," },
            { ar: "ٱلَّذِى يُوَسْوِسُ فِى صُدُورِ ٱلنَّاسِ", id: "yang membisikkan (kejahatan) ke dalam dada manusia," },
            { ar: "مِنَ ٱلْجِنَّةِ وَٱلنَّاسِ", id: "dari (golongan) jin dan manusia.'" },
        ]
    },
    { 
        id: 3, name: "Al-Falaq", ayat: 5, arabic: "قُلْ أَعُوذُ بِرَبِّ ٱلْفَلَقِ", translation: "Waktu Subuh", 
        verses: [
            { ar: "قُلْ أَعُوذُ بِرَبِّ ٱلْفَلَقِ", id: "Katakanlah (Muhammad), 'Aku berlindung kepada Tuhan yang menguasai subuh (fajar)," },
            { ar: "مِن شَرِّ مَا خَلَقَ", id: "dari kejahatan (makhluk yang) Dia ciptakan," },
            { ar: "وَمِن شَرِّ غَاسِقٍ إِذَا وَقَبَ", id: "dan dari kejahatan malam apabila telah gelap gulita," },
            { ar: "وَمِن شَرِّ ٱلنَّفَّٰثَٰتِ فِى ٱلْعُقَدِ", id: "dan dari kejahatan perempuan-perempuan penyihir yang meniup pada buhul-buhul (tali)," },
            { ar: "وَمِن شَرِّ حَاسِدٍ إِذَا حَسَدَ", id: "dan dari kejahatan orang yang dengki apabila dia dengki.'" },
        ]
    },
    { 
        id: 4, name: "Al-Ikhlas", ayat: 4, arabic: "قُلْ هُوَ ٱللَّهُ أَحَدٌ", translation: "Memurnikan Keesaan Allah", 
        verses: [
            { ar: "قُلْ هُوَ ٱللَّهُ أَحَدٌ", id: "Katakanlah (Muhammad), 'Dialah Allah, Yang Maha Esa." },
            { ar: "ٱللَّهُ ٱلصَّمَدُ", id: "Allah tempat meminta segala sesuatu." },
            { ar: "لَمْ يَلِدْ وَلَمْ يُولَدْ", id: "(Allah) tidak beranak dan tidak pula diperanakkan." },
            { ar: "وَلَمْ يَكُن لَّهُۥ كُفُوًا أَحَدٌ", id: "Dan tidak ada sesuatu yang setara dengan Dia.'" },
        ]
    },
    { 
        id: 5, name: "Al-Masad", ayat: 5, arabic: "تَبَّتْ يَدَآ أَبِى لَهَبٍ وَتَبَّ", translation: "Gejolak Api (Abu Lahab)", 
        verses: [
            { ar: "تَبَّتْ يَدَآ أَبِى لَهَبٍ وَتَبَّ", id: "Binasalah kedua tangan Abu Lahab dan benar-benar binasa dia." },
            { ar: "مَآ أَغْنَىٰ عَنْهُ مَالُهُۥ وَمَا كَسَبَ", id: "Tidaklah berguna baginya hartanya dan apa yang dia usahakan." },
            { ar: "سَيَصْلَىٰ نَارًا ذَاتَ لَهَبٍ", id: "Kelak dia akan masuk ke dalam api yang bergejolak (neraka)." },
            { ar: "وَٱمْرَأَتُهُۥ حَمَّالَةَ ٱلْحَطَبِ", id: "Dan (begitu pula) istrinya, pembawa kayu bakar (penyebar fitnah)." },
            { ar: "فِى جِيدِهَا حَبْلٌ مِّن مَّسَدٍ", id: "Di lehernya ada tali dari sabut yang dipintal." },
        ]
    },
    { 
        id: 6, name: "An-Nasr", ayat: 3, arabic: "إِذَا جَآءَ نَصْرُ ٱللَّهِ وَٱلْفَتْحُ", translation: "Pertolongan", 
        verses: [
            { ar: "إِذَا جَآءَ نَصْرُ ٱللَّهِ وَٱلْفَتْحُ", id: "Apabila telah datang pertolongan Allah dan kemenangan," },
            { ar: "وَرَأَيْتَ ٱلنَّاسَ يَدْخُلُونَ فِى دِينِ ٱللَّهِ أَفْوَاجًا", id: "dan engkau melihat manusia berbondong-bondong masuk agama Allah," },
            { ar: "فَسَبِّحْ بِحَمْدِ رَبِّكَ وَٱسْتَغْفِرْهُ ۚ إِنَّهُۥ كَانَ تَوَّابًۢا", id: "maka bertasbihlah dengan memuji Tuhanmu dan mohonlah ampunan kepada-Nya. Sungguh, Dia Maha Penerima tobat." },
        ]
    },
    { 
        id: 7, name: "Al-Kafirun", ayat: 6, arabic: "قُلْ يَٰٓأَيُّهَا ٱلْكَٰفِرُونَ", translation: "Orang-orang Kafir", 
        verses: [
            { ar: "قُلْ يَٰٓأَيُّهَا ٱلْكَٰفِرُونَ", id: "Katakanlah (Muhammad), 'Wahai orang-orang kafir!" },
            { ar: "لَآ أَعْبُدُ مَا تَعْبُدُونَ", id: "Aku tidak akan menyembah apa yang kamu sembah." },
            { ar: "وَلَآ أَنتُمْ عَٰبِدُونَ مَآ أَعْبُدُ", id: "Dan kamu bukan penyembah apa yang aku sembah." },
            { ar: "وَلَآ أَنَا۠ عَابِدٌ مَّا عَبَدتُّمْ", id: "Dan aku tidak pernah menjadi penyembah apa yang kamu sembah." },
            { ar: "وَلَآ أَنتُمْ عَٰبِدُونَ مَآ أَعْبُدُ", id: "Dan kamu tidak pernah (pula) menjadi penyembah apa yang aku sembah." },
            { ar: "لَكُمْ دِينُكُمْ وَلِىَ دِينِ", id: "Untukmu agamamu, dan untukku agamaku.'" },
        ]
    },
    { 
        id: 8, name: "Al-Kautsar", ayat: 3, arabic: "إِنَّآ أَعْطَيْنَٰكَ ٱلْكَوْثَرَ", translation: "Nikmat yang Banyak", 
        verses: [
            { ar: "إِنَّآ أَعْطَيْنَٰكَ ٱلْكَوْثَرَ", id: "Sungguh, Kami telah memberimu (Muhammad) nikmat yang banyak." },
            { ar: "فَصَلِّ لِرَبِّكَ وَٱنْحَرْ", id: "Maka laksanakanlah salat karena Tuhanmu, dan berkurbanlah (sebagai ibadah dan mendekatkan diri kepada Allah)." },
            { ar: "إِنَّ شَانِئَكَ هُوَ ٱلْأَبْتَرُ", id: "Sungguh, orang-orang yang membencimu dialah yang terputus (dari rahmat Allah)." },
        ]
    },
    { 
        id: 9, name: "Al-Ma'un", ayat: 7, arabic: "أَرَءَيْتَ ٱلَّذِى يُكَذِّبُ بِٱلدِّينِ", translation: "Barang-barang yang Berguna", 
        verses: [
            { ar: "أَرَءَيْتَ ٱلَّذِى يُكَذِّبُ بِٱلدِّينِ", id: "Tahukah kamu (orang) yang mendustakan agama?" },
            { ar: "فَذَٰلِكَ ٱلَّذِى يَدُعُّ ٱلْيَتِيمَ", id: "Maka itulah orang yang menghardik anak yatim," },
            { ar: "وَلَا يَحُضُّ عَلَىٰ طَعَامِ ٱلْمِسْكِينِ", id: "dan tidak mendorong memberi makan orang miskin." },
            { ar: "فَوَيْلٌ لِّلْمُصَلِّينَ", id: "Maka celakalah bagi orang-orang yang salat," },
            { ar: "ٱلَّذِينَ هُمْ عَن صَلَاتِهِمْ سَاهُونَ", id: "(yaitu) orang-orang yang lalai dari salatnya," },
            { ar: "ٱلَّذِينَ هُمْ يُرَآءُونَ", id: "orang-orang yang berbuat riya'," },
            { ar: "وَيَمْنَعُونَ ٱلْمَاعُونَ", id: "dan enggan (memberikan) bantuan." },
        ]
    },
    { 
        id: 10, name: "Quraisy", ayat: 4, arabic: "لِإِيلَٰفِ قُرَيْشٍ", translation: "Suku Quraisy", 
        verses: [
            { ar: "لِإِيلَٰفِ قُرَيْشٍ", id: "Karena kebiasaan orang-orang Quraisy," },
            { ar: "إِۦلَٰفِهِمْ رِحْلَةَ ٱلشِّتَآءِ وَٱلصَّيْفِ", id: "(yaitu) kebiasaan mereka bepergian pada musim dingin dan musim panas." },
            { ar: "فَلْيَعْبُدُوا۟ رَبَّ هَٰذَا ٱلْبَيْتِ", id: "Maka hendaklah mereka menyembah Tuhan (pemilik) rumah ini (Ka'bah)," },
            { ar: "ٱلَّذِىٓ أَطْعَمَهُم مِّن جُوعٍ وَءَامَنَهُم مِّنْ خَوْفٍۭ", id: "yang telah memberi makan kepada mereka untuk menghilangkan lapar dan mengamankan mereka dari rasa ketakutan." },
        ]
    },
    { 
        id: 11, name: "Al-Fil", ayat: 5, arabic: "أَلَمْ تَرَ كَيْفَ فَعَلَ رَبُّكَ بِأَصْحَٰبِ ٱلْفِيلِ", translation: "Gajah", 
        verses: [
            { ar: "أَلَمْ تَرَ كَيْفَ فَعَلَ رَبُّكَ بِأَصْحَٰبِ ٱلْفِيلِ", id: "Tidakkah engkau (Muhammad) perhatikan, bagaimana Tuhanmu telah bertindak terhadap pasukan bergajah?" },
            { ar: "أَلَمْ يَجْعَلْ كَيْدَهُمْ فِى تَضْلِيلٍ", id: "Bukankah Dia telah menjadikan tipu daya mereka (untuk menghancurkan Ka'bah) sia-sia?" },
            { ar: "وَأَرْسَلَ عَلَيْهِمْ طَيْرًا أَبَابِيلَ", id: "Dan Dia mengirimkan kepada mereka burung yang berbondong-bondong," },
            { ar: "تَرْمِيهِم بِحِجَارَةٍ مِّن سِجِّيلٍ", id: "yang melempari mereka dengan batu (berasal) dari tanah liat yang dibakar," },
            { ar: "فَجَعَلَهُمْ كَعَصْفٍ مَّأْكُولٍ", id: "sehingga mereka dijadikan-Nya seperti daun-daun yang dimakan (ulat)." },
        ]
    },
    { 
        id: 12, name: "Al-Humazah", ayat: 9, arabic: "وَيْلٌ لِّكُلِّ هُمَزَةٍ لُّمَزَةٍ", translation: "Pengumpat", 
        verses: [
            { ar: "وَيْلٌ لِّكُلِّ هُمَزَةٍ لُّمَزَةٍ", id: "Celakalah bagi setiap pengumpat dan pencela," },
            { ar: "ٱلَّذِى جَمَعَ مَالًا وَعَدَّدَهُۥ", id: "yang mengumpulkan harta dan menghitung-hitungnya," },
            { ar: "يَحْسَبُ أَنَّ مَالَهُۥٓ أَخْلَدَهُۥ", id: "dia (manusia) mengira bahwa hartanya itu dapat mengekalkannya." },
            { ar: "كَلَّا ۖ لَيُنۢبَذَنَّ فِى ٱلْحُطَمَةِ", id: "Sekali-kali tidak! Pasti dia akan dilemparkan ke dalam Hutamah." },
            { ar: "وَمَآ أَدْرَىٰكَ مَا ٱلْحُطَمَةُ", id: "Dan tahukah kamu apakah Hutamah itu?" },
            { ar: "نَارُ ٱللَّهِ ٱلْمُوقَدَةُ", id: "(Yaitu) api (azab) Allah yang dinyalakan," },
            { ar: "ٱلَّتِى تَطَّلِعُ عَلَى ٱلْأَفْـِٔدَةِ", id: "yang (membakar) sampai ke hati." },
            { ar: "إِنَّهَا عَلَيْهِم مُّؤْصَدَةٌ", id: "Sungguh, api itu ditutup rapat atas (diri) mereka," },
            { ar: "فِى عَمَدٍ مُّمَدَّدَةٍۭ", id: "(sedang mereka itu) diikat pada tiang-tiang yang panjang." },
        ]
    },
    { 
        id: 13, name: "Al-Asr", ayat: 3, arabic: "وَٱلْعَصْرِ", translation: "Masa", 
        verses: [
            { ar: "وَٱلْعَصْرِ", id: "Demi masa." },
            { ar: "إِنَّ ٱلْإِنسَٰنَ لَفِى خُسْرٍ", id: "Sungguh, manusia berada dalam kerugian," },
            { ar: "إِلَّا ٱلَّذِينَ ءَامَنُوا۟ وَعَمِلُوا۟ ٱلصَّٰلِحَٰتِ وَتَوَاصَوْا۟ بِٱلْحَقِّ وَتَوَاصَوْا۟ بِٱلصَّبْرِ", id: "kecuali orang-orang yang beriman dan mengerjakan kebajikan serta saling menasihati untuk kebenaran dan saling menasihati untuk kesabaran." },
        ]
    },
    { 
        id: 14, name: "At-Takatsur", ayat: 8, arabic: "أَلْهَىٰكُمُ ٱلتَّكَاثُرُ", translation: "Bermegah-megahan", 
        verses: [
            { ar: "أَلْهَىٰكُمُ ٱلتَّكَاثُرُ", id: "Bermegah-megahan telah melalaikan kamu," },
            { ar: "حَتَّىٰ زُرْتُمُ ٱلْمَقَابِرَ", id: "sampai kamu masuk ke dalam kubur." },
            { ar: "كَلَّا سَوْفَ تَعْلَمُونَ", id: "Sekali-kali tidak! Kelak kamu akan mengetahui (akibat perbuatanmu itu)." },
            { ar: "ثُمَّ كَلَّا سَوْفَ تَعْلَمُونَ", id: "Kemudian sekali-kali tidak! Kelak kamu akan mengetahui." },
            { ar: "كَلَّا لَوْ تَعْلَمُونَ عِلْمَ ٱلْيَقِينِ", id: "Sekali-kali tidak! Sekiranya kamu mengetahui dengan pasti." },
            { ar: "لَتَرَوُنَّ ٱلْجَحِيمَ", id: "Niscaya kamu benar-benar akan melihat neraka Jahim." },
            { ar: "ثُمَّ لَتَرَوُنَّهَا عَيْنَ ٱلْيَقِينِ", id: "Kemudian kamu benar-benar akan melihatnya dengan mata kepala sendiri." },
            { ar: "ثُمَّ لَتُسْـَٔلُنَّ يَوْمَئِذٍ عَنِ ٱلنَّعِيمِ", id: "Kemudian kamu benar-benar akan ditanya pada hari itu tentang kenikmatan (yang megah di dunia itu)." },
        ]
    },
    { 
        id: 15, name: "Al-Qari'ah", ayat: 11, arabic: "ٱلْقَارِعَةُ", translation: "Hari Kiamat", 
        verses: [
            { ar: "ٱلْقَارِعَةُ", id: "Hari Kiamat," },
            { ar: "مَا ٱلْقَارِعَةُ", id: "apakah hari Kiamat itu?" },
            { ar: "وَمَآ أَدْرَىٰكَ مَا ٱلْقَارِعَةُ", id: "Dan tahukah kamu apakah hari Kiamat itu?" },
            { ar: "يَوْمَ يَكُونُ ٱلنَّاسُ كَٱلْفَرَاشِ ٱلْمَبْثُوثِ", id: "Pada hari itu manusia seperti laron yang berterbangan," },
            { ar: "وَتَكُونُ ٱلْجِبَالُ كَٱلْعِهْنِ ٱلْمَنفُوشِ", id: "dan gunung-gunung seperti bulu yang dihambur-hamburkan." },
            { ar: "فَأَمَّا مَن ثَقُلَتْ مَوَٰزِينُهُۥ", id: "Maka adapun orang yang berat timbangan (kebaikan)nya," },
            { ar: "فَهُوَ فِى عِيشَةٍ رَّاضِيَةٍ", id: "maka dia berada dalam kehidupan yang memuaskan." },
            { ar: "وَأَمَّا مَنْ خَفَّتْ مَوَٰزِينُهُۥ", id: "Dan adapun orang yang ringan timbangan (kebaikan)nya," },
            { ar: "فَأُمُّهُۥ هَاوِيَةٌ", id: "maka tempat kembalinya adalah neraka Hawiyah." },
            { ar: "وَمَآ أَدْرَىٰكَ مَا هِيَهْ", id: "Dan tahukah kamu apakah neraka Hawiyah itu?" },
            { ar: "نَارٌ حَامِيَةٌ", id: "(Yaitu) api yang sangat panas." },
        ]
    },
    // Data dummy terstruktur untuk surah sisanya
    { id: 16, name: "Al-'Adiyat", ayat: 11, arabic: "وَٱلْعَٰدِيَٰتِ ضَبْحًا", translation: "Kuda Perang yang Berlari Kencang", 
        verses: [
            { ar: "وَٱلْعَٰدِيَٰتِ ضَبْحًا", id: "Demi kuda perang yang berlari kencang terengah-engah," },
            { ar: "فَٱلْمُورِيَٰتِ قَدْحًا", id: "dan kuda yang memercikkan bunga api (dengan pukulan tapak kakinya)," },
            { ar: "فَٱلْمُغِيرَٰتِ صُبْحًا", id: "dan kuda yang menyerang (dengan tiba-tiba) pada waktu pagi," },
            { ar: "فَأَثَرْنَ بِهِۦ نَقْعًا", id: "sehingga menerbangkan debu," },
            { ar: "فَوَسَطْنَ بِهِۦ جَمْعًا", id: "lalu menyerbu ke tengah-tengah kumpulan (musuh)," },
            { ar: "إِنَّ ٱلْإِنسَٰنَ لِرَبِّهِۦ لَكَنُودٌ", id: "sungguh, manusia itu sangat ingkar, tidak berterima kasih kepada Tuhannya," },
            { ar: "وَإِنَّهُۥ عَلَىٰ ذَٰلِكَ لَشَهِيدٌ", id: "dan sesungguhnya dia (manusia) menyaksikan (mengakui) keingkarannya." },
            { ar: "وَإِنَّهُۥ لِحُبِّ ٱلْخَيْرِ لَشَدِيدٌ", id: "Dan sesungguhnya cintanya kepada harta benar-benar berlebihan." },
            { ar: "أَفَلَا يَعْلَمُ إِذَا بُعْثِرَ مَا فِى ٱلْقُبُورِ", id: "Maka tidakkah dia mengetahui apabila apa yang di dalam kubur dikeluarkan?" },
            { ar: "وَحُصِّلَ مَا فِى ٱلصُّدُورِ", id: "dan apa yang tersimpan di dalam dada dilahirkan?" },
            { ar: "إِنَّ رَبَّهُم بِهِمْ يَوْمَئِذٍ لَّخَبِيرٌ", id: "Sungguh, Tuhan mereka pada hari itu Maha Mengetahui keadaan mereka." },
        ]
    },
    { id: 17, name: "Az-Zalzalah", ayat: 8, arabic: "إِذَا زُلْزِلَتِ ٱلْأَرْضُ زِلْزَالَهَا", translation: "Goncangan",
        verses: [
            { ar: "إِذَا زُلْزِلَتِ ٱلْأَرْضُ زِلْزَالَهَا", id: "Apabila bumi digoncangkan dengan goncangan yang dahsyat," },
            { ar: "وَأَخْرَجَتِ ٱلْأَرْضُ أَثْقَالَهَا", id: "dan bumi telah mengeluarkan beban-beban berat (yang dikandung)nya," },
            { ar: "وَقَالَ ٱلْإِنسَٰنُ مَا لَهَا", id: "dan manusia bertanya, 'Apa yang terjadi pada bumi ini?'" },
            { ar: "يَوْمَئِذٍ تُحَدِّثُ أَخْبَارَهَا", id: "Pada hari itu bumi menyampaikan beritanya," },
            { ar: "بِأَنَّ رَبَّكَ أَوْحَىٰ لَهَا", id: "karena sesungguhnya Tuhanmu telah memerintahkan (yang sedemikian itu) padanya." },
            { ar: "يَوْمَئِذٍ يَصْدُرُ ٱلنَّاسُ أَشْتَاتًا لِّيُرَوْا۟ أَعْمَٰلَهُمْ", id: "Pada hari itu manusia keluar dari kuburnya dalam keadaan berkelompok-kelompok, untuk diperlihatkan kepada mereka (balasan) semua perbuatannya." },
            { ar: "فَمَن يَعْمَلْ مِثْقَالَ ذَرَّةٍ خَيْرًا يَرَهُۥ", id: "Maka barangsiapa mengerjakan kebaikan seberat zarah, niscaya dia akan melihat (balasan)nya." },
            { ar: "وَمَن يَعْمَلْ مِثْقَالَ ذَرَّةٍ شَرًّا يَرَهُۥ", id: "Dan barangsiapa mengerjakan kejahatan seberat zarah, niscaya dia akan melihat (balasan)nya." },
        ]
    },
    { id: 18, name: "Al-Bayyinah", ayat: 8, arabic: "لَمْ يَكُنِ ٱلَّذِينَ كَفَرُوا۟ مِنْ أَهْلِ ٱلْكِتَٰبِ", translation: "Bukti Nyata",
        verses: [
            { ar: "لَمْ يَكُنِ ٱلَّذِينَ كَفَرُوا۟ مِنْ أَهْلِ ٱلْكِتَٰبِ وَٱلْمُشْرِكِينَ مُنفَكِّينَ حَتَّىٰ تَأْتِيَهُمُ ٱلْبَيِّنَةُ", id: "Orang-orang kafir dari golongan Ahli Kitab dan orang-orang musyrik tidak akan meninggalkan (agama mereka) sampai datang kepada mereka bukti yang nyata," },
            { ar: "رَسُولٌ مِّنَ ٱللَّهِ يَتْلُوا۟ صُحُفًا مُّطَهَّرَةً", id: "(yaitu) seorang Rasul dari Allah (Muhammad) yang membacakan lembaran-lembaran yang suci (Al-Qur'an)," },
            { ar: "فِيهَا كُتُبٌ قَيِّمَةٌ", id: "di dalamnya terdapat (isi) yang lurus (benar)." },
            { ar: "وَمَا تَفَرَّقَ ٱلَّذِينَ أُوتُوا۟ ٱلْكِتَٰبَ إِلَّا مِنۢ بَعْدِ مَا جَآءَتْهُمُ ٱلْبَيِّنَةُ", id: "Dan tidaklah terpecah belah orang-orang yang didatangkan Al-Kitab (kepada mereka) melainkan setelah datang kepada mereka bukti yang nyata." },
            { ar: "وَمَآ أُمِرُوٓا۟ إِلَّا لِيَعْبُدُوا۟ ٱللَّهَ مُخْلِصِينَ لَهُ ٱلدِّينَ حُنَفَآءَ وَيُقِيمُوا۟ ٱلصَّلَوٰةَ وَيُؤْتُوا۟ ٱلزَّكَوٰةَ ۚ وَذَٰلِكَ دِينُ ٱلْقَيِّمَةِ", id: "Padahal mereka hanya diperintah menyembah Allah dengan ikhlas menaati-Nya semata-mata karena (menjalankan) agama, dan juga agar melaksanakan salat dan menunaikan zakat; dan yang demikian itulah agama yang lurus (benar)." },
            { ar: "إِنَّ ٱلَّذِينَ كَفَرُوا۟ مِنْ أَهْلِ ٱلْكِتَٰبِ وَٱلْمُشْرِكِينَ فِى نَارِ جَهَنَّمَ خَٰلِدِينَ فِيهَآ ۚ أُو۟لَٰٓئِكَ هُمْ شَرُّ ٱلْبَرِيَّةِ", id: "Sungguh, orang-orang kafir dari golongan Ahli Kitab dan orang-orang musyrik (akan masuk) ke neraka Jahanam; mereka kekal di dalamnya. Mereka itu adalah sejahat-jahat makhluk." },
            { ar: "إِنَّ ٱلَّذِينَ ءَامَنُوا۟ وَعَمِلُوا۟ ٱلصَّٰلِحَٰتِ أُو۟لَٰٓئِكَ هُمْ خَيْرُ ٱلْبَرِيَّةِ", id: "Sungguh, orang-orang yang beriman dan mengerjakan kebajikan, mereka itu adalah sebaik-baik makhluk." },
            { ar: "جَزَآؤُهُمْ عِندَ رَبِّهِمْ جَنَّٰتُ عَدْنٍ تَجْرِى مِن تَحْتِهَا ٱلْأَنْهَٰرُ خَٰلِدِينَ فِيهَآ أَبَدًا ۖ رَّضِىَ ٱللَّهُ عَنْهُمْ وَرَضُوا۟ عَنْهُ ۚ ذَٰلِكَ لِمَن خَشِىَ رَبَّهُۥ", id: "Balasan mereka di sisi Tuhan mereka ialah surga 'Adn yang mengalir di bawahnya sungai-sungai; mereka kekal di dalamnya selama-lamanya. Allah rida terhadap mereka dan mereka pun rida kepada-Nya. Itulah (balasan) bagi orang yang takut kepada Tuhannya." },
        ]
    },
    { id: 19, name: "Al-Qadr", ayat: 5, arabic: "إِنَّآ أَنزَلْنَٰهُ فِى لَيْلَةِ ٱلْقَدْرِ", translation: "Kemuliaan",
        verses: [
            { ar: "إِنَّآ أَنزَلْنَٰهُ فِى لَيْلَةِ ٱلْقَدْرِ", id: "Sesungguhnya Kami telah menurunkannya (Al-Qur'an) pada malam kemuliaan." },
            { ar: "وَمَآ أَدْرَىٰكَ مَا لَيْلَةُ ٱلْقَدْرِ", id: "Dan tahukah kamu apakah malam kemuliaan itu?" },
            { ar: "لَيْلَةُ ٱلْقَدْرِ خَيْرٌ مِّنْ أَلْفِ شَهْرٍ", id: "Malam kemuliaan itu lebih baik daripada seribu bulan." },
            { ar: "تَنَزَّلُ ٱلْمَلَٰٓئِكَةُ وَٱلرُّوحُ فِيهَا بِإِذْنِ رَبِّهِم مِّن كُلِّ أَمْرٍ", id: "Pada malam itu turun para malaikat dan Rūh (Jibril) dengan izin Tuhannya untuk mengatur semua urusan." },
            { ar: "سَلَٰمٌ هِىَ حَتَّىٰ مَطْلَعِ ٱلْفَجْرِ", id: "Sejahteralah (malam itu) sampai terbit fajar." },
        ]
    },
    { id: 20, name: "Al-'Alaq", ayat: 19, arabic: "ٱقْرَأْ بِٱسْمِ رَبِّكَ ٱلَّذِى خَلَقَ", translation: "Segumpal Darah",
        verses: [
            { ar: "ٱقْرَأْ بِٱٱسْمِ رَبِّكَ ٱلَّذِى خَلَقَ", id: "Bacalah dengan (menyebut) nama Tuhanmu yang menciptakan," },
            { ar: "خَلَقَ ٱلْإِنسَٰنَ مِنْ عَلَقٍ", id: "Dia telah menciptakan manusia dari segumpal darah." },
            { ar: "ٱقْرَأْ وَرَبُّكَ ٱلْأَكْرَمُ", id: "Bacalah, dan Tuhanmulah Yang Maha Mulia," },
            { ar: "ٱلَّذِى عَلَّمَ بِٱلْقَلَمِ", id: "Yang mengajar (manusia) dengan pena." },
            { ar: "عَلَّمَ ٱلْإِنسَٰنَ مَا لَمْ يَعْلَمْ", id: "Dia mengajarkan manusia apa yang tidak diketahuinya." },
            { ar: "كَلَّآ إِنَّ ٱلْإِنسَٰنَ لَيَطْغَىٰٓ", id: "Sekali-kali tidak! Sungguh, manusia benar-benar melampaui batas," },
            { ar: "أَن رَّءَاهُ ٱسْتَغْنَىٰٓ", id: "apabila melihat dirinya serba cukup." },
            { ar: "إِنَّ إِلَىٰ رَبِّكَ ٱلرُّجْعَىٰٓ", id: "Sungguh, hanya kepada Tuhanmulah tempat kembali(mu)." },
            { ar: "أَرَءَيْتَ ٱلَّذِى يَنْهَىٰ", id: "Bagaimana pendapatmu tentang orang yang melarang," },
            { ar: "عَبْدًا إِذَا صَلَّىٰٓ", id: "seorang hamba ketika dia melaksanakan salat," },
            { ar: "أَرَءَيْتَ إِن كَانَ عَلَى ٱلْهُدَىٰٓ", id: "Bagaimana pendapatmu jika dia (yang dilarang salat) berada di atas kebenaran (petunjuk)," },
            { ar: "أَوْ أَمَرَ بِٱلتَّقْوَىٰٓ", id: "atau dia menyuruh bertakwa (kepada Allah)?" },
            { ar: "أَرَءَيْتَ إِن كَذَّبَ وَتَوَلَّىٰٓ", id: "Bagaimana pendapatmu jika dia (yang melarang) itu mendustakan dan berpaling?" },
            { ar: "أَلَمْ يَعْلَم بِأَنَّ ٱللَّهَ يَرَىٰ", id: "Tidakkah dia mengetahui bahwa sesungguhnya Allah melihat (segala perbuatannya)?" },
            { ar: "كَلَّا لَئِن لَّمْ يَنتَهِ لَنَسْفَعًا بِٱلنَّاصِيَةِ", id: "Sekali-kali tidak! Sungguh, jika dia tidak berhenti niscaya Kami tarik ubun-ubunnya," },
            { ar: "نَاصِيَةٍ كَٰذِبَةٍ خَاطِئَةٍ", id: "(yaitu) ubun-ubun orang yang mendustakan lagi durhaka." },
            { ar: "فَلْيَدْعُ نَادِيَهُۥ", id: "Maka biarlah dia memanggil golongannya (untuk menolongnya)," },
            { ar: "سَنَدْعُ ٱلزَّبَانِيَةَ", id: "Kelak Kami akan memanggil Malaikat Zabaniyah (penyiksa)," },
            { ar: "كَلَّا لَا تُطِعْهُ وَٱسْجُدْ وَٱقْتَرِبْ ۩", id: "Sekali-kali tidak! Janganlah kamu patuh kepadanya; dan sujudlah serta dekatkanlah (dirimu kepada Allah)." },
        ]
    },
    { id: 21, name: "At-Tin", ayat: 8, arabic: "وَٱلتِّينِ وَٱلزَّيْتُونِ", translation: "Buah Tin",
        verses: [
            { ar: "وَٱلتِّينِ وَٱٱلزَّيْتُونِ", id: "Demi (buah) tin dan (buah) zaitun," },
            { ar: "وَطُورِ سِينِينَ", id: "demi gunung Sinai," },
            { ar: "وَهَٰذَا ٱلْبَلَدِ ٱلْأَمِينِ", id: "dan demi negeri (Mekah) yang aman ini." },
            { ar: "لَقَدْ خَلَقْنَا ٱلْإِنسَٰنَ فِىٓ أَحْسَنِ تَقْوِيمٍ", id: "Sungguh, Kami telah menciptakan manusia dalam bentuk yang sebaik-baiknya," },
            { ar: "ثُمَّ رَدَدْنَٰهُ أَسْفَلَ سَٰفِلِينَ", id: "kemudian Kami kembalikan dia ke tempat yang serendah-rendahnya (neraka)," },
            { ar: "إِلَّا ٱلَّذِينَ ءَامَنُوا۟ وَعَمِلُوا۟ ٱلصَّٰلِحَٰتِ فَلَهُمْ أَجْرٌ غَيْرُ مَمْنُونٍ", id: "kecuali orang-orang yang beriman dan mengerjakan kebajikan; maka mereka akan mendapat pahala yang tidak putus-putusnya." },
            { ar: "فَمَا يُكَذِّبُكَ بَعْدُ بِٱلدِّينِ", id: "Maka apa yang menyebabkan kamu mendustakan (hari) pembalasan setelah (adanya keterangan-keterangan) itu?" },
            { ar: "أَلَيْسَ ٱللَّهُ بِأَحْكَمِ ٱلْحَٰكِمِينَ", id: "Bukankah Allah hakim yang paling adil?" },
        ]
    },
    { id: 22, name: "Asy-Syarh", ayat: 8, arabic: "أَلَمْ نَشْرَحْ لَكَ صَدْرَكَ", translation: "Kelapangan", 
        verses: [
            { ar: "أَلَمْ نَشْرَحْ لَكَ صَدْرَكَ", id: "Bukankah Kami telah melapangkan dadamu (Muhammad)," },
            { ar: "وَوَضَعْنَا عَنكَ وِزْرَكَ", id: "dan Kami telah menghilangkan darimu bebanmu," },
            { ar: "ٱلَّذِىٓ أَنقَضَ ظَهْرَكَ", id: "yang memberatkan punggungmu?" },
            { ar: "وَرَفَعْنَا لَكَ ذِكْرَكَ", id: "Dan Kami tinggikan sebutan (nama)mu bagimu." },
            { ar: "فَإِنَّ مَعَ ٱلْعُسْرِ يُسْرًا", id: "Maka sesungguhnya bersama kesulitan ada kemudahan," },
            { ar: "إِنَّ مَعَ ٱلْعُسْرِ يُسْرًا", id: "sesungguhnya bersama kesulitan ada kemudahan." },
            { ar: "فَإِذَا فَرَغْتَ فَٱنصَبْ", id: "Maka apabila engkau telah selesai (dari sesuatu urusan), tetaplah bekerja keras (untuk urusan yang lain)," },
            { ar: "وَإِلَىٰ رَبِّكَ فَٱرْغَب", id: "dan hanya kepada Tuhanmulah engkau berharap." },
        ]
    },
    { id: 23, name: "Ad-Dhuha", ayat: 11, arabic: "وَٱلضُّحَىٰ", translation: "Waktu Dhuha",
        verses: [
            { ar: "وَٱٱلضُّحَىٰ", id: "Demi waktu duha (ketika matahari naik sepenggalah)," },
            { ar: "وَٱٱلَّيْلِ إِذَا سَجَىٰ", id: "dan demi malam apabila telah sunyi." },
            { ar: "مَا وَدَّعَكَ رَبُّكَ وَمَا قَلَىٰ", id: "Tuhanmu (Muhammad) tidak meninggalkan engkau dan tidak (pula) membencimu," },
            { ar: "وَلَلْءَاخِرَةُ خَيْرٌ لَّكَ مِنَ ٱلْأُولَىٰ", id: "dan sungguh, yang kemudian itu lebih baik bagimu daripada yang permulaan." },
            { ar: "وَلَسَوْفَ يُعْطِيكَ رَبُّكَ فَتَرْضَىٰٓ", id: "Dan sungguh, kelak Tuhanmu pasti memberikan karunia-Nya kepadamu, sehingga engkau menjadi puas." },
            { ar: "أَلَمْ يَجِدْكَ يَتِيمًا فَـَٔاوَىٰ", id: "Bukankah Dia mendapatimu sebagai seorang yatim, lalu Dia melindungi(mu)?" },
            { ar: "وَوَجَدَكَ ضَآلًّا فَهَدَىٰ", id: "Dan Dia mendapatimu sebagai seorang yang bingung, lalu Dia memberikan petunjuk?" },
            { ar: "وَوَجَدَكَ عَٰٓئِلًا فَأَغْنَىٰ", id: "Dan Dia mendapatimu sebagai seorang yang kekurangan, lalu Dia memberikan kecukupan?" },
            { ar: "فَأَمَّا ٱلْيَتِيمَ فَلَا تَقْهَرْ", id: "Maka terhadap anak yatim janganlah engkau berlaku sewenang-wenang." },
            { ar: "وَأَمَّا ٱٱلسَّآئِلَ فَلَا تَنْهَرْ", id: "Dan terhadap orang yang meminta-minta janganlah engkau menghardik." },
            { ar: "وَأَمَّا بِنِعْمَةِ رَبِّكَ فَحَدِّثْ", id: "Dan terhadap nikmat Tuhanmu, hendaklah engkau nyatakan (dengan bersyukur)." },
        ]
    },
    { id: 24, name: "Al-Lail", ayat: 21, arabic: "وَٱلَّيْلِ إِذَا يَغْشَىٰ", translation: "Malam",
        verses: [
            { ar: "وَٱٱلَّيْلِ إِذَا يَغْشَىٰ", id: "Demi malam apabila menutupi (cahaya siang)," },
            { ar: "وَٱٱلنَّهَارِ إِذَا تَجَلَّىٰ", id: "demi siang apabila terang benderang," },
            { ar: "وَمَا خَلَقَ ٱٱلذَّكَرَ وَٱٱلْأُنثَىٰٓ", id: "dan demi penciptaan laki-laki dan perempuan." },
            { ar: "إِنَّ سَعْيَكُمْ لَشَتَّىٰ", id: "Sungguh, usaha kamu memang beraneka macam." },
            { ar: "فَأَمَّا مَن أَعْطَىٰ وَٱتَّقَىٰ", id: "Maka barangsiapa memberikan (hartanya di jalan Allah) dan bertakwa," },
            { ar: "وَصَدَّقَ بِٱٱلْحُسْنَىٰ", id: "dan membenarkan (adanya pahala) yang terbaik (surga)," },
            { ar: "فَسَنُيَسِّرُهُۥ لِلْيُسْرَىٰ", id: "maka Kami akan melapangkan baginya jalan kemudahan (kebahagiaan)." },
            { ar: "وَأَمَّا مَنۢ بَخِلَ وَٱسْتَغْنَىٰ", id: "Dan adapun orang yang kikir dan merasa dirinya cukup (tidak perlu pertolongan Allah)," },
            { ar: "وَكَذَّبَ بِٱٱلْحُسْنَىٰ", id: "serta mendustakan (adanya pahala) yang terbaik," },
            { ar: "فَسَنُيَسِّرُهُۥ لِلْعُسْرَىٰ", id: "maka akan Kami mudahkan baginya jalan kesukaran (kesengsaraan)," },
            { ar: "وَمَا يُغْنِى عَنْهُ مَالُهُۥٓ إِذَا تَرَدَّىٰٓ", id: "dan hartanya tidak bermanfaat baginya apabila dia telah binasa." },
            { ar: "إِنَّ عَلَيْنَا لَلْهُدَىٰ", id: "Sesungguhnya kewajiban Kamilah memberi petunjuk," },
            { ar: "وَإِنَّ لَنَا لَلْءَاخِرَةَ وَٱٱلْأُولَىٰ", id: "dan sesungguhnya milik Kamilah akhirat dan dunia." },
            { ar: "فَأَنذَرْتُكُمْ نَارًا تَلَظَّىٰ", id: "Maka Aku peringatkan kamu dengan api yang menyala-nyala (neraka)," },
            { ar: "لَا يَصْلَىٰهَآ إِلَّا ٱلْأَشْقَى", id: "yang tidak ada yang masuk ke dalamnya kecuali orang yang paling celaka," },
            { ar: "ٱٱلَّذِى كَذَّبَ وَتَوَلَّىٰ", id: "yang mendustakan (kebenaran) dan berpaling (dari keimanan)." },
            { ar: "وَسَيُجَنَّبُهَا ٱلْأَتْقَى", id: "Dan akan dijauhkan darinya (neraka) orang yang paling bertakwa," },
            { ar: "ٱٱلَّذِى يُؤْتِى مَالَهُۥ يَتَزَكَّىٰ", id: "yang menginfakkan hartanya (di jalan Allah) untuk membersihkan (dirinya)," },
            { ar: "وَمَا لِأَحَدٍ عِندَهُۥ مِن نِّعْمَةٍ تُجْزَىٰٓ", id: "dan tidak ada seorang pun memberikan suatu nikmat kepadanya yang harus dibalasnya," },
            { ar: "إِلَّا ٱبْتِغَآءَ وَجْهِ رَبِّهِ ٱلْأَعْلَىٰ", id: "kecuali (dia memberikan itu semata-mata) karena mencari keridaan Tuhannya Yang Mahatinggi." },
            { ar: "وَلَسَوْفَ يَرْضَىٰ", id: "Dan sungguh, kelak dia akan mendapat kepuasan." },
        ]
    },
    { id: 25, name: "Asy-Syams", ayat: 15, arabic: "وَٱلشَّمْسِ وَضُحَىٰهَا", translation: "Matahari",
        verses: [
            { ar: "وَٱٱلشَّمْسِ وَضُحَىٰهَا", id: "Demi matahari dan cahayanya di pagi hari," },
            { ar: "وَٱٱلْقَمَرِ إِذَا تَلَىٰهَا", id: "demi bulan apabila mengiringinya," },
            { ar: "وَٱٱلنَّهَارِ إِذَا جَلَّىٰهَا", id: "demi siang apabila menampakkannya," },
            { ar: "وَٱٱلَّيْلِ إِذَا يَغْشَىٰهَا", id: "demi malam apabila menutupinya (gelap gulita)," },
            { ar: "وَٱٱلسَّمَآءِ وَمَا بَنَىٰهَا", id: "demi langit serta pembinaannya (yang menakjubkan)," },
            { ar: "وَٱٱلْأَرْضِ وَمَا طَحَىٰهَا", id: "demi bumi serta penghamparannya," },
            { ar: "وَنَفْسٍ وَمَا سَوَّىٰهَا", id: "demi jiwa serta penyempurnaan (ciptaan)nya," },
            { ar: "فَأَلْهَمَهَا فُجُورَهَا وَتَقْوَىٰهَا", id: "maka Dia mengilhamkan kepadanya (jalan) kejahatan dan ketakwaannya," },
            { ar: "قَدْ أَفْلَحَ مَن زَكَّىٰهَا", id: "sungguh beruntung orang yang menyucikannya (jiwa itu)," },
            { ar: "وَقَدْ خَابَ مَن دَسَّىٰهَا", id: "dan sungguh rugi orang yang mengotorinya." },
            { ar: "كَذَّبَتْ ثَمُودُ بِطَغْوَىٰهَآ", id: "(Kaum) Samud telah mendustakan (rasulnya) karena kedurhakaannya," },
            { ar: "إِذِ ٱنۢبَعَثَ أَشْقَىٰهَا", id: "ketika bangkit orang yang paling celaka di antara mereka," },
            { ar: "فَقَالَ لَهُمْ رَسُولُ ٱللَّهِ نَاقَةَ ٱللَّهِ وَسُقْيَٰهَا", id: "lalu Rasul Allah (Saleh) berkata kepada mereka, “Tinggalkanlah unta betina Allah dan minumlah.”" },
            { ar: "فَكَذَّبُوهُ فَعَقَرُوهَا فَدَمْدَمَ عَلَيْهِمْ رَبُّهُم بِذَنۢبِهِمْ فَسَوَّىٰهَا", id: "Namun, mereka mendustakannya dan menyembelihnya, karena itu Tuhan menimpakan azab atas mereka karena dosa mereka, lalu diratakan-Nya (dengan tanah)." },
            { ar: "وَلَا يَخَافُ عُقْبَٰهَا", id: "Dan Dia tidak takut terhadap akibatnya." },
        ]
    },
    { id: 26, name: "Al-Balad", ayat: 20, arabic: "لَآ أُقْسِمُ بِهَٰذَا ٱلْبَلَدِ", translation: "Negeri",
        verses: [
            { ar: "لَآ أُقْسِمُ بِهَٰذَا ٱلْبَلَدِ", id: "Aku bersumpah dengan negeri ini (Mekah)," },
            { ar: "وَأَنتَ حِلٌّۢ بِهَٰذَا ٱلْبَلَدِ", id: "dan engkau (Muhammad) bertempat di negeri (Mekah) ini," },
            { ar: "وَوَالِدٍ وَمَا وَلَدَ", id: "dan demi bapak serta anaknya." },
            { ar: "لَقَدْ خَلَقْنَا ٱلْإِنسَٰنَ فِى كَبَدٍ", id: "Sungguh, Kami telah menciptakan manusia berada dalam susah payah." },
            { ar: "أَيَحْسَبُ أَن لَّن يَقْدِرَ عَلَيْهِ أَحَدٌ", id: "Apakah dia (manusia) mengira bahwa tidak ada sesuatu pun yang berkuasa atasnya?" },
            { ar: "يَقُولُ أَهْلَكْتُ مَالًا لُّبَدًا", id: "Dia mengatakan, “Aku telah menghabiskan harta yang banyak.”" },
            { ar: "أَيَحْسَبُ أَن لَّمْ يَرَهُۥٓ أَحَدٌ", id: "Apakah dia mengira bahwa tidak ada seorang pun yang melihatnya?" },
            { ar: "أَلَمْ نَجْعَل لَّهُۥ عَيْنَيْنِ", id: "Bukankah Kami telah menjadikan untuknya sepasang mata," },
            { ar: "وَلِسَانًا وَشَفَتَيْنِ", id: "lidah, dan dua bibir." },
            { ar: "وَهَدَيْنَٰهُ ٱٱلنَّجْدَيْنِ", id: "Dan Kami telah menunjukkan kepadanya dua jalan kebaikan dan kejahatan?" },
            { ar: "فَلَا ٱقْتَحَمَ ٱلْعَقَبَةَ", id: "Tetapi dia tidak menempuh jalan yang mendaki dan sukar." },
            { ar: "وَمَآ أَدْرَىٰكَ مَا ٱلْعَقَبَةُ", id: "Dan tahukah kamu apakah jalan yang mendaki dan sukar itu?" },
            { ar: "فَكُّ رَقَبَةٍ", id: "(Yaitu) melepaskan perbudakan (memerdekakan hamba sahaya)," },
            { ar: "أَوْ إِطْعَٰمٌ فِى يَوْمٍ ذِى مَسْغَبَةٍ", id: "atau memberi makan pada hari terjadi kelaparan," },
            { ar: "يَتِيمًا ذَا مَقْرَبَةٍ", id: "(kepada) anak yatim yang memiliki hubungan kerabat," },
            { ar: "أَوْ مِسْكِينًا ذَا مَتْرَبَةٍ", id: "atau orang miskin yang sangat fakir." },
            { ar: "ثُمَّ كَانَ مِنَ ٱلَّذِينَ ءَامَنُوا۟ وَتَوَاصَوْا۟ بِٱٱلصَّبْرِ وَتَوَاصَوْا۟ بِٱٱلْمَرْحَمَةِ", id: "Kemudian dia termasuk orang-orang yang beriman dan saling menasihati untuk bersabar dan saling menasihati untuk berkasih sayang." },
            { ar: "أُو۟لَٰٓئِكَ أَصْحَٰبُ ٱلْمَيْمَنَةِ", id: "Mereka (orang-orang yang beriman dan saling menasihati) itu adalah golongan kanan." },
            { ar: "وَٱٱلَّذِينَ كَفَرُوا۟ بِـَٔايَٰتِنَا هُمْ أَصْحَٰبُ ٱلْمَشْـَٔمَةِ", id: "Dan orang-orang yang kafir kepada ayat-ayat Kami, mereka itu adalah golongan kiri." },
            { ar: "عَلَيْهِمْ نَارٌ مُّؤْصَدَةٌ", id: "Mereka berada dalam api neraka yang ditutup rapat." },
        ]
    },
    { id: 27, name: "Al-Fajr", ayat: 30, arabic: "وَٱلْفَجْرِ", translation: "Fajar",
        verses: [
            { ar: "وَٱٱلْفَجْرِ", id: "Demi fajar," },
            { ar: "وَلَيَالٍ عَشْرٍ", id: "demi malam yang sepuluh," },
            { ar: "وَٱٱلشَّفْعِ وَٱٱلْوَتْرِ", id: "demi yang genap dan yang ganjil," },
            { ar: "وَٱٱلَّيْلِ إِذَا يَسْرِ", id: "demi malam apabila berlalu." },
            { ar: "هَلْ فِى ذَٰلِكَ قَسَمٌ لِّذِى حِجْرٍ", id: "Apakah pada yang demikian itu terdapat sumpah (yang dapat diterima) bagi orang yang berakal?" },
            { ar: "أَلَمْ تَرَ كَيْفَ فَعَلَ رَبُّكَ بِعَادٍ", id: "Tidakkah engkau perhatikan bagaimana Tuhanmu berbuat terhadap kaum 'Ad?" },
            { ar: "إِرَمَ ذَاتِ ٱلْعِمَادِ", id: "(yaitu) penduduk Iram (ibu kota kaum 'Ad) yang mempunyai bangunan-bangunan yang tinggi," },
            { ar: "ٱٱلَّتِى لَمْ يُخْلَقْ مِثْلُهَا فِى ٱلْبِلَٰدِ", id: "yang belum pernah diciptakan (suatu kota) seperti itu di negeri-negeri lain," },
            { ar: "وَثَمُودَ ٱٱلَّذِينَ جَابُوا۟ ٱٱلصَّخْرَ بِٱٱلْوَادِ", id: "dan kaum Samud yang memotong batu-batu besar di lembah," },
            { ar: "وَفِرْعَوْنَ ذِى ٱلْأَوْتَادِ", id: "dan Fir'aun yang mempunyai pasak-pasak (bangunan piramida yang besar)," },
            { ar: "ٱٱلَّذِينَ طَغَوْا۟ فِى ٱٱلْبِلَٰدِ", id: "yang berbuat sewenang-wenang dalam negeri," },
            { ar: "فَأَكْثَرُوا۟ فِيهَا ٱٱلْفَسَادَ", id: "lalu mereka banyak berbuat kerusakan dalam negeri itu," },
            { ar: "فَصَبَّ عَلَيْهِمْ رَبُّكَ سَوْطَ عَذَابٍ", id: "karena itu Tuhanmu menimpakan cemeti azab kepada mereka." },
            { ar: "إِنَّ رَبَّكَ لَلَبِٱلْمِرْصَادِ", id: "Sungguh, Tuhanmu benar-benar mengawasi." },
            { ar: "فَأَمَّا ٱٱلْإِنسَٰنُ إِذَا مَا ٱبْتَلَىٰهُ رَبُّهُۥ فَأَكْرَمَهُۥ وَنَعَّمَهُۥ فَيَقُولُ رَبِّىٓ أَكْرَمَنِ", id: "Maka adapun manusia, apabila Tuhan mengujinya lalu memuliakannya dan memberinya kesenangan, maka dia berkata, 'Tuhanku telah memuliakanku.'" },
            { ar: "وَأَمَّآ إِذَا مَا ٱبْتَلَىٰهُ فَقَدَرَ عَلَيْهِ رِزْقَهُۥ فَيَقُولُ رَبِّىٓ أَهَٰنَنِ", id: "Namun, apabila Tuhan mengujinya lalu membatasi rezekinya, maka dia berkata, 'Tuhanku menghinaku.'" },
            { ar: "كَلَّا ۖ بَل لَّا تُكْرِمُونَ ٱٱلْيَتِيمَ", id: "Sekali-kali tidak! Bahkan kamu tidak memuliakan anak yatim," },
            { ar: "وَلَا تَحَٰٓضُّونَ عَلَىٰ طَعَامِ ٱٱلْمِسْكِينِ", id: "dan kamu tidak saling mengajak memberi makan orang miskin," },
            { ar: "وَتَأْكُلُونَ ٱٱلتُّرَاثَ أَكْلًا لَّمًّا", id: "sedangkan kamu memakan harta warisan dengan cara mencampurbaurkan (yang halal dan yang haram)," },
            { ar: "وَتُحِبُّونَ ٱٱلْمَالَ حُبًّا جَمًّا", id: "dan kamu mencintai harta dengan kecintaan yang berlebihan." },
            { ar: "كَلَّآ إِذَا دُكَّتِ ٱٱلْأَرْضُ دَكًّا دَكًّا", id: "Sekali-kali tidak! Apabila bumi diguncangkan berturut-turut," },
            { ar: "وَجَآءَ رَبُّكَ وَٱٱلْمَلَكُ صَفًّا صَفًّا", id: "dan datanglah Tuhanmu; sedang malaikat berbaris-baris." },
            { ar: "وَجِا۟ىٓءَ يَوْمَئِذٍۭ بِجَهَنَّمَ ۚ يَوْمَئِذٍ يَتَذَكَّرُ ٱٱلْإِنسَٰنُ وَأَنَّىٰ لَهُ ٱٱلذِّكْرَىٰ", id: "Dan pada hari itu diperlihatkan neraka Jahanam; pada hari itu sadarlah manusia, tetapi tidak berguna lagi kesadaran itu baginya." },
            { ar: "يَقُولُ يَٰلَيْتَنِى قَدَّمْتُ لِحَيَاتِى", id: "Dia berkata, 'Alangkah baiknya kalau dahulu aku mengerjakan (kebajikan) untuk hidupku ini.'" },
            { ar: "فَيَوْمَئِذٍ لَّا يُعَذِّبُ عَذَابَهُۥٓ أَحَدٌ", id: "Maka pada hari itu tidak ada seorang pun yang mengazab seperti azab-Nya (Allah)," },
            { ar: "وَلَا يُوثِقُ وَثَاقَهُۥٓ أَحَدٌ", id: "dan tidak ada seorang pun yang mengikat seperti ikatan-Nya." },
            { ar: "يَٰٓأَيَّتُهَا ٱٱلنَّفْسُ ٱٱلْمُطْمَئِنَّةُ", id: "Wahai jiwa yang tenang!" },
            { ar: "ٱرْجِعِىٓ إِلَىٰ رَبِّكِ رَاضِيَةً مَّرْضِيَّةً", id: "Kembalilah kepada Tuhanmu dengan hati yang rida dan diridai-Nya." },
            { ar: "فَٱٱدْخُلِى فِى عِبَٰدِى", id: "Maka masuklah ke dalam golongan hamba-hamba-Ku," },
            { ar: "وَٱٱدْخُلِى جَنَّتِى", id: "dan masuklah ke dalam surga-Ku." },
        ]
    },
    { id: 28, name: "Al-Ghasyiyah", ayat: 26, arabic: "هَلْ أَتَىٰكَ حَدِيثُ ٱلْغَٰشِيَةِ", translation: "Hari Pembalasan",
        verses: [
            { ar: "هَلْ أَتَىٰكَ حَدِيثُ ٱٱلْغَٰشِيَةِ", id: "Sudahkah sampai kepadamu berita tentang hari Kiamat?" },
            { ar: "وُجُوهٌ يَوْمَئِذٍ خَٰشِعَةٌ", id: "Pada hari itu banyak wajah yang tertunduk hina," },
            { ar: "عَامِلَةٌ نَّاصِبَةٌ", id: "bekerja keras lagi kepayahan," },
            { ar: "تَصْلَىٰ نَارًا حَامِيَةً", id: "mereka masuk ke dalam api yang sangat panas (neraka)," },
            { ar: "تُسْقَىٰ مِنْ عَيْنٍ ءَانِيَةٍ", id: "diberi minum dari sumber air yang sangat panas." },
            { ar: "لَّيْسَ لَهُمْ طَعَامٌ إِلَّا مِن ضَرِيعٍ", id: "Tidak ada makanan bagi mereka selain dari pohon yang berduri," },
            { ar: "لَّا يُسْمِنُ وَلَا يُغْنِى مِن جُوعٍ", id: "yang tidak menggemukkan dan tidak menghilangkan lapar." },
            { ar: "وُجُوهٌ يَوْمَئِذٍ نَّاعِمَةٌ", id: "Pada hari itu banyak (pula) wajah yang berseri-seri," },
            { ar: "لِّسَعْيِهَا رَاضِيَةٌ", id: "merasa puas karena usahanya," },
            { ar: "فِى جَنَّةٍ عَالِيَةٍ", id: "di dalam surga yang tinggi," },
            { ar: "لَّا تَسْمَعُ فِيهَا لَٰغِيَةً", id: "di sana kamu tidak mendengar perkataan yang tidak berguna." },
            { ar: "فِيهَا عَيْنٌ جَارِيَةٌ", id: "Di sana ada mata air yang mengalir." },
            { ar: "فِيهَا سُرُرٌ مَّرْفُوعَةٌ", id: "Di sana ada dipan-dipan yang ditinggikan," },
            { ar: "وَأَكْوَابٌ مَّوْضُوعَةٌ", id: "dan gelas-gelas yang tersedia (di dekatnya)," },
            { ar: "وَنَمَارِقُ مَصْفُوفَةٌ", id: "dan bantal-bantal sandaran yang tersusun," },
            { ar: "وَزَرَابِىُّ مَبْثُوثَةٌ", id: "dan permadani-permadani yang terhampar." },
            { ar: "أَفَلَا يَنظُرُونَ إِلَى ٱٱلْإِبِلِ كَيْفَ خُلِقَتْ", id: "Maka tidakkah mereka memperhatikan unta bagaimana diciptakan?" },
            { ar: "وَإِلَى ٱٱلسَّمَآءِ كَيْفَ رُفِعَتْ", id: "Dan langit, bagaimana ditinggikan?" },
            { ar: "وَإِلَى ٱٱلْجِبَالِ كَيْفَ نُصِبَتْ", id: "Dan gunung-gunung bagaimana ditegakkan?" },
            { ar: "وَإِلَى ٱٱلْأَرْضِ كَيْفَ سُطِحَتْ", id: "Dan bumi, bagaimana dihamparkan?" },
            { ar: "فَذَكِّرْ إِنَّمَآ أَنتَ مُذَكِّرٌ", id: "Maka berilah peringatan, karena sesungguhnya engkau (Muhammad) hanyalah pemberi peringatan." },
            { ar: "لَّسْتَ عَلَيْهِم بِمُصَيْطِرٍ", id: "Engkau bukanlah orang yang berkuasa atas mereka," },
            { ar: "إِلَّا مَن تَوَلَّىٰ وَكَفَرَ", id: "tetapi barangsiapa berpaling dan menjadi kafir," },
            { ar: "فَيُعَذِّبُهُ ٱٱللَّهُ ٱٱلْعَذَابَ ٱٱلْأَكْبَرَ", id: "maka Allah akan mengazabnya dengan azab yang terbesar." },
            { ar: "إِنَّ إِلَيْنَآ إِيَابَهُمْ", id: "Sungguh, kepada Kami-lah kembali mereka," },
            { ar: "ثُمَّ إِنَّ عَلَيْنَا حِسَابَهُم", id: "kemudian sungguh, Kami yang akan menghisab mereka." },
        ]
    },
    { id: 29, name: "Al-A'la", ayat: 19, arabic: "سَبِّحِ ٱسْمَ رَبِّكَ ٱلْأَعْلَى", translation: "Yang Paling Tinggi",
        verses: [
            { ar: "سَبِّحِ ٱٱسْمَ رَبِّكَ ٱٱلْأَعْلَى", id: "Sucikanlah nama Tuhanmu Yang Mahatinggi," },
            { ar: "ٱٱلَّذِى خَلَقَ فَسَوَّىٰ", id: "yang menciptakan, lalu menyempurnakan (ciptaan-Nya)," },
            { ar: "وَٱٱلَّذِى قَدَّرَ فَهَدَىٰ", id: "yang menentukan kadar (masing-masing) dan memberi petunjuk," },
            { ar: "وَٱٱلَّذِىٓ أَخْرَجَ ٱٱلْمَرْعَىٰ", id: "dan yang menumbuhkan rerumputan," },
            { ar: "فَجَعَلَهُۥ غُثَآءً أَحْوَىٰ", id: "lalu dijadikan-Nya (rumput-rumput) itu kering kehitam-hitaman." },
            { ar: "سَنُقْرِئُكَ فَلَا تَنسَىٰٓ", id: "Kami akan membacakan (Al-Qur'an) kepadamu (Muhammad) sehingga engkau tidak akan lupa," },
            { ar: "إِلَّا مَا شَآءَ ٱٱللَّهُ ۚ إِنَّهُۥ يَعْلَمُ ٱٱلْجَهْرَ وَمَا يَخْفَىٰ", id: "kecuali jika Allah menghendaki. Sungguh, Dia mengetahui yang terang dan yang tersembunyi." },
            { ar: "وَنُيَسِّرُكَ لِلْيُسْرَىٰ", id: "Dan Kami akan melapangkan bagimu jalan kemudahan (menuju kebahagiaan)." },
            { ar: "فَذَكِّرْ إِن نَّفَعَتِ ٱٱلذِّكْرَىٰ", id: "Oleh sebab itu berikanlah peringatan, karena peringatan itu bermanfaat," },
            { ar: "سَيَذَّكَّرُ مَن يَخْشَىٰ", id: "orang yang takut (kepada Allah) akan mendapat pelajaran," },
            { ar: "وَيَتَجَنَّبُهَا ٱٱلْأَشْقَى", id: "dan orang yang celaka (kafir) akan menjauhinya," },
            { ar: "ٱٱلَّذِى يَصْلَى ٱٱلنَّارَ ٱٱلْكُبْرَىٰ", id: "(yaitu) orang yang akan memasuki api yang besar (neraka)." },
            { ar: "ثُمَّ لَا يَمُوتُ فِيهَا وَلَا يَحْيَىٰ", id: "Kemudian dia tidak mati di dalamnya dan tidak (pula) hidup." },
            { ar: "قَدْ أَفْلَحَ مَن تَزَكَّىٰ", id: "Sungguh beruntung orang yang menyucikan diri (dengan beriman)," },
            { ar: "وَذَكَرَ ٱٱسْمَ رَبِّهِۦ فَصَلَّىٰ", id: "dan dia ingat nama Tuhannya, lalu dia salat." },
            { ar: "بَلْ تُؤْثِرُونَ ٱٱلْحَيَوٰةَ ٱٱلدُّنْيَا", id: "Sedangkan kamu (orang-orang kafir) memilih kehidupan dunia," },
            { ar: "وَٱٱلْءَاخِرَةُ خَيْرٌ وَأَبْقَىٰٓ", id: "padahal kehidupan akhirat itu lebih baik dan lebih kekal." },
            { ar: "إِنَّ هَٰذَا لَفِى ٱٱلصُّحُفِ ٱٱلْأُولَىٰ", id: "Sesungguhnya ini terdapat dalam kitab-kitab yang dahulu," },
            { ar: "صُحُفِ إِبْرَٰهِيمَ وَمُوسَىٰ", id: "(yaitu) Kitab-kitab Ibrahim dan Musa." },
        ]
    },
    { id: 30, name: "At-Tariq", ayat: 17, arabic: "وَٱلسَّمَآءِ وَٱلطَّارِقِ", translation: "Yang Datang di Malam Hari",
        verses: [
            { ar: "وَٱٱلسَّمَآءِ وَٱٱلطَّارِقِ", id: "Demi langit dan yang datang pada malam hari." },
            { ar: "وَمَآ أَدْرَىٰكَ مَا ٱٱلطَّارِقُ", id: "Dan tahukah kamu apakah yang datang pada malam hari itu?" },
            { ar: "ٱٱلنَّجْمُ ٱٱلثَّاقِبُ", id: "(Yaitu) bintang yang bersinar tajam," },
            { ar: "إِن كُلُّ نَفْسٍ لَّمَّا عَلَيْهَا حَافِظٌ", id: "setiap orang pasti ada penjaganya." },
            { ar: "فَلْيَنظُرِ ٱٱلْإِنسَٰنُ مِمَّ خُلِقَ", id: "Maka hendaklah manusia memperhatikan dari apa dia diciptakan." },
            { ar: "خُلِقَ مِن مَّآءٍ دَافِقٍ", id: "Dia diciptakan dari air yang memancar," },
            { ar: "يَخْرُجُ مِنۢ بَيْنِ ٱٱلصُّلْبِ وَٱٱلتَّرَآئِبِ", id: "yang keluar dari antara tulang sulbi (punggung) dan tulang dada." },
            { ar: "إِنَّهُۥ عَلَىٰ رَجْعِهِۦ لَقَادِرٌ", id: "Sungguh, Dia (Allah) benar-benar kuasa untuk mengembalikannya (hidup kembali)." },
            { ar: "يَوْمَ تُبْلَى ٱٱلسَّرَآئِرُ", id: "Pada hari ditampakkan segala rahasia," },
            { ar: "فَمَا لَهُۥ مِن قُوَّةٍ وَلَا نَاصِرٍ", id: "maka manusia tidak lagi mempunyai suatu kekuatan dan tidak (pula) ada penolong." },
            { ar: "وَٱٱلسَّمَآءِ ذَاتِ ٱٱلرَّجْعِ", id: "Demi langit yang mengandung hujan," },
            { ar: "وَٱٱلْأَرْضِ ذَاتِ ٱٱلصَّدْعِ", id: "dan bumi yang mempunyai tumbuh-tumbuhan," },
            { ar: "إِنَّهُۥ لَقَوْلٌ فَصْلٌ", id: "sungguh, (Al-Qur'an) itu benar-benar firman yang memisahkan antara yang hak dan yang batil," },
            { ar: "وَمَا هُوَ بِٱٱلْهَزْلِ", id: "dan (Al-Qur'an) itu bukanlah sendagurauan." },
            { ar: "إِنَّهُمْ يَكِيدُونَ كَيْدًا", id: "Sungguh, mereka (orang kafir) merencanakan tipu daya yang jahat," },
            { ar: "وَأَكِيدُ كَيْدًا", id: "dan Aku pun membuat rencana (tipu daya) yang jitu." },
            { ar: "فَمَهِّلِ ٱٱلْكَٰفِرِينَ أَمْهِلْهُمْ رُوَيْدًا", id: "Karena itu berilah tenggang waktu kepada orang-orang kafir itu. Beri tangguhlah mereka sebentar." },
        ]
    },
];

const DAILY_DHIKR = {
    0: { day: 'Minggu', arabic: 'يَا حَيُّ يَا قَيُّومُ', translation: 'Wahai Yang Maha Hidup, Wahai Yang Berdiri Sendiri', benefit: 'Mendapat kemuliaan dan kemudahan dalam urusan.' },
    1: { day: 'Senin', arabic: 'لَا حَوْلَ وَلَا قُوَّةَ إِلَّا بِٱللَّهِ', translation: 'Tiada daya dan tiada kekuatan kecuali dengan pertolongan Allah', benefit: 'Dihindarkan dari kemudaratan dan dimudahkan rezeki.' },
    2: { day: 'Selasa', arabic: 'سُبْحَانَ ٱللَّهِ وَبِحَمْدِهِ', translation: 'Maha Suci Allah dengan segala puji bagi-Nya', benefit: 'Dihapuskan dosa-dosa walaupun sebanyak buih di lautan.' },
    3: { day: 'Rabu', arabic: 'أَسْتَغْفِرُ ٱللَّهَ ٱلْعَظِيمَ', translation: 'Aku memohon ampun kepada Allah Yang Maha Agung', benefit: 'Diampuni dosa dan mendapat ketenangan jiwa.' },
    4: { day: 'Kamis', arabic: 'سُبْحَانَ ٱللَّهِ ٱلْعَظِيمِ وَبِحَمْدِهِ', translation: 'Maha Suci Allah Yang Maha Agung dan dengan segala puji bagi-Nya', benefit: 'Ditanamkan satu pohon kurma di syurga.' },
    5: { day: 'Jumat', arabic: 'اللَّهُمَّ صَلِّ عَلَى مُحَمَّدٍ', translation: 'Ya Allah, berikan rahmat kepada Nabi Muhammad', benefit: 'Mendapat syafaat Nabi Muhammad SAW di Hari Kiamat.' },
    6: { day: 'Sabtu', arabic: 'لَا إِلَٰهَ إِلَّا ٱللَّهُ', translation: 'Tiada Tuhan selain Allah', benefit: 'Membuka pintu rezeki dan kebahagiaan.' },
};

const getTodaysDhikr = () => {
    const todayIndex = new Date().getDay(); 
    return DAILY_DHIKR[todayIndex];
};

export default function App() {
    // --- STATE MANAGEMENT ---
    const [db, setDb] = useState(null); 
    const [userId, setUserId] = useState(null);
    const [isLoading, setIsLoading] = useState(true);
    // State untuk Navigasi (Tab Utama)
    const [activeTab, setActiveTab] = useState('Home'); 
    // State untuk Navigasi Quran (Sub-halaman)
    const [quranView, setQuranView] = useState({ page: 'list', selectedSurah: null }); 
    
    const [prayerTimes, setPrayerTimes] = useState([]);
    const [prayerRecords, setPrayerRecords] = useState({});
    const [isTrackingLoading, setIsTrackingLoading] = useState(false);
    const [haidData, setHaidData] = useState({});
    
    // --- DUMMY DATA ---
    const userName = "Fallenaru";
    const location = "Malang, Indonesia";
    const today = new Date();
    const dateGregorian = today.toLocaleDateString('id-ID', { year: 'numeric', month: 'long', day: 'numeric', weekday: 'long' });
    const dateHijri = "11 Rajab 1447"; 
    
    // --- CALCULATED STATUSES ---
    const haidStatus = useMemo(() => getHaidStatus(haidData), [haidData]);
    const todaysDhikr = useMemo(() => getTodaysDhikr(), []);

    const activePrayerData = useMemo(() => {
        if (prayerTimes.length === 0) return null;
        const nowMs = new Date().getTime();
        const sortedPrayers = [...prayerTimes]; 
        let activePrayer = null;
        let nextPrayer = null;

        for (let i = sortedPrayers.length - 1; i >= 0; i--) {
            const prayer = sortedPrayers[i];
            const prayerStartMs = getPrayerDate(prayer.time).getTime();
            
            if (nowMs >= prayerStartMs) {
                activePrayer = prayer;
                nextPrayer = sortedPrayers[i + 1];
                break;
            }
        }

        if (activePrayer) {
            if (!nextPrayer) {
                nextPrayer = sortedPrayers.find(p => p.key === 'Fajr');
            }
            
            const startTime12hr = formatTime12h(activePrayer.time);
            const endTime12hr = formatTime12h(nextPrayer ? nextPrayer.time : '04:00'); 
            
            const startTimeDisplay = startTime12hr.replace(';', ':');
            const endTimeDisplay = endTime12hr.replace(';', ':');
            
            return {
                ...activePrayer,
                timeRange: `${startTimeDisplay} - ${endTimeDisplay}`,
                isDone: !!prayerRecords[activePrayer.key],
            };
        }
        
        const fajr = sortedPrayers.find(p => p.key === 'Fajr');
        const dhuhr = sortedPrayers.find(p => p.key === 'Dhuhr');
        if (fajr) {
            const startTime12hr = formatTime12h(fajr.time);
            const endTime12hr = formatTime12h(dhuhr ? dhuhr.time : '08:00'); 
            
            const startTimeDisplay = startTime12hr.replace(':', ';');
            const endTimeDisplay = endTime12hr.replace(':', ';');
            
            return {
                ...fajr,
                timeRange: `Menuju ${startTimeDisplay}`, 
                isDone: !!prayerRecords[fajr.key],
                isUpcoming: true,
                name: "Subuh (Berikutnya)"
            };
        }

        return null;
    }, [prayerTimes, prayerRecords]);
    
    // --- FIREBASE SETUP ---
    useEffect(() => {
        // PERUBAHAN UNTUK GITHUB: Jika config tidak ada, jangan hentikan app,
        // tapi set isLoading false agar UI tetap tampil dengan data dummy.
        if (Object.keys(firebaseConfig).length === 0) {
            console.warn("Firebase config not found. Running in offline/demo mode.");
            setIsLoading(false);
            return () => {};
        }

        const app = initializeApp(firebaseConfig);
        const firestore = getFirestore(app);
        const firebaseAuth = getAuth(app);
        
        setDb(firestore); 

        let unsubRecords = () => {};
        let unsubHaid = () => {};

        const authenticate = async () => {
            if (firebaseAuth.currentUser) return; 

            try {
                if (initialAuthToken) {
                    await signInWithCustomToken(firebaseAuth, initialAuthToken);
                } else {
                    await signInAnonymously(firebaseAuth);
                }
            } catch (error) {
                console.error("Auth error:", error);
                // Fallback jika auth gagal, biarkan app berjalan
                setIsLoading(false);
            }
        };

        const unsubscribeAuth = onAuthStateChanged(firebaseAuth, (user) => {
            unsubRecords();
            unsubHaid();

            if (user) {
                setUserId(user.uid);
                setIsLoading(false);

                const todayDocId = new Date().toISOString().slice(0, 10); 
              
                const userRecordsCollectionRef = collection(firestore, `artifacts/${appId}/users/${user.uid}/prayer_records`);
                const todayDocRef = doc(userRecordsCollectionRef, todayDocId);

                unsubRecords = onSnapshot(todayDocRef, (docSnapshot) => {
                    setPrayerRecords(docSnapshot.exists() ? docSnapshot.data() : {});
                }, (error) => console.error("Records error:", error));

                const haidDocRef = doc(firestore, `artifacts/${appId}/users/${user.uid}/settings/haid_tracker`);
                unsubHaid = onSnapshot(haidDocRef, (docSnapshot) => {
                    setHaidData(docSnapshot.exists() ? docSnapshot.data() : {});
                }, (error) => console.error("Haid error:", error));
              
            } else {
                setUserId(null); 
                authenticate();
            }
        });

        return () => {
            unsubscribeAuth();
            unsubRecords();
            unsubHaid();
        };
    }, []); 
    
    const saveHaidData = async (data) => {
        if (!db || !userId) return; 
        const haidDocRef = doc(db, `artifacts/${appId}/users/${userId}/settings/haid_tracker`);
        try {
            await setDoc(haidDocRef, data, { merge: true });
        } catch (error) {
            console.error("Save haid error:", error);
        }
    };

    const trackPrayer = async (prayerKey) => {
        if (!db || !userId || isTrackingLoading || haidStatus.isHaid) return; 
        setIsTrackingLoading(true);

        const todayDocId = new Date().toISOString().slice(0, 10);
        const userRecordsCollectionRef = collection(db, `artifacts/${appId}/users/${userId}/prayer_records`);
        const todayDocRef = doc(userRecordsCollectionRef, todayDocId);

        try {
            await setDoc(todayDocRef, { 
                [prayerKey]: true, 
                updatedAt: new Date().toISOString() 
            }, { merge: true });
        } catch (error) {
            console.error("Track prayer error:", error);
        } finally {
            setIsTrackingLoading(false);
        }
    };

    const fetchPrayerTimes = useCallback(() => {
        // Dummy times for Jakarta/Western Indonesia
        const realPrayerTimes = [
            { name: "Subuh", time: '04:35', key: 'Fajr' },
            { name: "Dzuhur", time: '12:05', key: 'Dhuhr' },
            { name: "Ashar", time: '15:15', key: 'Asr' }, 
            { name: "Maghrib", time: '18:10', key: 'Maghrib' },
            { name: "Isya", time: '19:25', key: 'Isha' }
        ];
        setPrayerTimes(realPrayerTimes);
    }, []);

    useEffect(() => {
        fetchPrayerTimes();
    }, [fetchPrayerTimes]);

    // --- COMPONENT RENDERERS ---

    const NavItem = ({ children, isActive, label, onClick }) => (
        <button 
            className={`flex flex-col items-center p-1 transition-colors focus:outline-none active:scale-95 ${isActive ? 'text-teal-600' : 'text-gray-400 hover:text-gray-500'}`}
            aria-label={label}
            onClick={onClick}
        >
            <span className="w-6 h-6 mb-0.5">{children}</span>
            <div className="text-xs font-medium">{label}</div>
        </button>
    );

    const ActivePrayerCard = ({ prayerData, haidStatus }) => {
        if (!prayerData) return (
            <div className="bg-amber-100 p-4 rounded-xl shadow-lg text-center animate-pulse">
                <p className="font-semibold text-amber-700">Memuat waktu salat...</p>
            </div>
        );

        let statusMessage = '';
        let buttonText = '';
        let buttonAction = () => {};
        let buttonDisabled = false;

        if (haidStatus.isHaid) {
            statusMessage = `Haid (H${haidStatus.haidDay}): Salat ditangguhkan.`;
            buttonText = 'Moon Mode Aktif';
            buttonDisabled = true;
        } else if (prayerData.isDone) {
            statusMessage = `Alhamdulillah! Salat ${prayerData.name} telah ditunaikan.`;
            buttonText = 'Selesai';
            buttonDisabled = true;
        } else {
            statusMessage = `Sekarang Waktu ${prayerData.name}. Segera tunaikan salat!`;
            buttonText = 'Tandai Selesai';
            buttonAction = () => trackPrayer(prayerData.key);
            buttonDisabled = isTrackingLoading;
        }

        return (
            <div className="relative bg-white rounded-xl overflow-hidden shadow-2xl transition-all hover:scale-[1.01] duration-300">
                <ThematicBackground />
                <div className="relative p-6 md:p-8 text-white z-10 space-y-4">
                    <p className="text-sm font-light uppercase tracking-widest opacity-80">Waktu Salat Saat Ini</p>
                    <h2 className="text-3xl md:text-5xl font-extrabold drop-shadow-lg">{prayerData.name}</h2>
                    <p className="text-lg md:text-xl font-medium flex items-center gap-2">
                        <svg className="w-5 h-5" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"><path d="M10 2v3a2 2 0 0 0 2 2h4a2 2 0 0 0 2-2V2"/><path d="M4 22h16"/><path d="M4 11h16"/><path d="M12 11V2"/><path d="M12 22V11"/></svg>
                        {prayerData.time.replace(':', ':')} WIB
                    </p>
                    <p className="text-xs italic opacity-90">{prayerData.timeRange}</p>

                    <div className="pt-4 flex flex-col sm:flex-row items-start sm:items-center justify-between gap-3">
                        <p className={`text-sm md:text-lg font-bold ${haidStatus.isHaid ? 'text-pink-100' : 'text-green-100'}`}>
                            {statusMessage}
                        </p>
                        <button
                            onClick={buttonAction}
                            disabled={buttonDisabled}
                            className={`px-4 py-2 text-sm font-bold rounded-full transition-all duration-300 shadow-xl min-w-[150px] whitespace-nowrap ${
                                haidStatus.isHaid
                                    ? 'bg-pink-700 text-pink-100 cursor-not-allowed disabled:opacity-80'
                                    : prayerData.isDone
                                        ? 'bg-green-600 text-white cursor-default'
                                        : 'bg-white text-amber-700 hover:bg-gray-100 active:scale-95 disabled:opacity-50'
                            }`}
                        >
                            {buttonText}
                        </button>
                    </div>
                </div>
            </div>
        );
    };
    
    const DailyDhikrCard = ({ dhikrData }) => {
        return (
            <div className="bg-white p-5 rounded-xl shadow-lg border-t-4 border-teal-500/50">
                <h3 className="text-xl font-extrabold text-teal-800 flex items-center mb-3">
                    <SparkleIcon className="w-5 h-5 mr-2 text-teal-600 fill-yellow-400"/> Dzikir Hari {dhikrData.day}
                </h3>
              
                <div className="bg-teal-50 p-4 rounded-lg">
                    <p className="text-2xl font-serif text-right leading-relaxed text-teal-900 mb-2">
                        {dhikrData.arabic}
                    </p>
                    <p className="text-sm font-medium text-teal-700">
                        {dhikrData.translation}
                    </p>
                </div>
              
                <p className="mt-3 text-xs italic text-gray-500">
                    Keutamaan: {dhikrData.benefit}
                </p>
            </div>
        );
    };

    const PrayerListItem = ({ prayer }) => {
        const isDone = !!prayerRecords[prayer.key];
        const hasPassed = isTimePassed(prayer.time);
        const isEarly = isWithinEarlyWindow(prayer.time);
        const isHaid = haidStatus.isHaid;

        let statusText = 'Akan Datang';
        let statusColor = 'text-gray-500';
        let buttonDisabled = isDone || isHaid;

        if (isHaid) {
            statusText = `Ditangguhkan`;
            statusColor = 'text-pink-600 font-bold';
            buttonDisabled = true;
        } else if (isDone) {
            statusText = 'Ditunaikan'; 
            statusColor = 'text-green-600 font-bold';
        } else if (hasPassed) {
            statusText = 'Terlewat';
            statusColor = 'text-red-500 font-bold';
            buttonDisabled = true; 
        } else if (isEarly) {
            statusText = 'Waktu Qabliyyah';
            statusColor = 'text-blue-500 font-bold';
            buttonDisabled = isTrackingLoading;
        } else {
            buttonDisabled = true; 
        }

        return (
            <div className="flex items-center justify-between p-4 bg-white rounded-xl shadow-md border-l-4 border-amber-500/20 mb-3 transition-shadow hover:shadow-lg">
                <div className="flex flex-col">
                    <span className="text-lg font-bold text-gray-800">{prayer.name}</span>
                    <span className="text-sm text-amber-600 font-semibold">{prayer.time.replace(':', ':')} WIB</span>
                </div>
                <div className="flex items-center gap-3">
                    <span className={`text-xs sm:text-sm ${statusColor} text-right hidden sm:block`}>
                        {statusText}
                    </span>
                    <button 
                        onClick={() => trackPrayer(prayer.key)}
                        disabled={buttonDisabled}
                        className={`w-10 h-10 flex items-center justify-center rounded-full transition-all duration-200 shadow-sm ${
                            isDone 
                                ? 'bg-green-100 text-green-600' 
                                : isHaid 
                                    ? 'bg-pink-50 text-pink-300 cursor-not-allowed'
                                    : hasPassed
                                        ? 'bg-gray-100 text-gray-300 cursor-not-allowed'
                                        : !buttonDisabled 
                                            ? 'bg-amber-100 text-amber-600 hover:bg-amber-200 active:scale-95 border border-amber-200'
                                            : 'bg-gray-50 text-gray-300 cursor-not-allowed'
                        }`}
                    >
                        {isDone ? <CheckCircleIcon className="w-6 h-6" /> : <div className={`w-5 h-5 rounded-full border-2 ${!buttonDisabled ? 'border-amber-400' : 'border-gray-300'}`} />}
                    </button>
                </div>
            </div>
        );
    };

    // --- QURAN SUB-VIEWS ---

    // Halaman Detail Surah - DIUBAH untuk menampilkan per-ayat
    const SurahDetailView = ({ surah, onBack }) => {
        if (!surah.verses || surah.verses.length === 0) {
            return (
                <div className="space-y-6 p-4 text-center">
                    <header className="flex items-center gap-3 border-b border-gray-200 pb-3">
                        <ArrowLeft className="w-6 h-6 text-gray-600 cursor-pointer hover:text-teal-600" onClick={onBack} />
                        <h1 className="text-xl font-bold text-gray-800 flex-1">{surah.name}</h1>
                    </header>
                    <p className="text-gray-500">Data ayat untuk surah ini tidak ditemukan.</p>
                </div>
            );
        }

        return (
            <div className="space-y-6 pb-4">
                {/* Header Surah */}
                <header className="flex flex-col gap-3 sticky top-0 bg-gray-50 py-3 border-b border-gray-100 z-20 shadow-sm">
                    <div className="flex items-center gap-3">
                        <ArrowLeft className="w-6 h-6 text-gray-600 cursor-pointer hover:text-teal-600 active:scale-90" onClick={onBack} />
                        <h1 className="text-xl font-bold text-gray-800 flex-1">{surah.name}</h1>
                    </div>
                    {/* Info Card - Hanya menampilkan Bismillah atau Ayat pertama sebagai penanda */}
                    <div className="bg-white p-3 rounded-lg shadow-sm text-center border-t border-teal-500/50 mx-4">
                        <p className="text-xs text-gray-500">{surah.translation} ({surah.ayat} Ayat)</p>
                        <p className="text-2xl font-serif text-teal-800 mt-1" dir="rtl">{surah.arabic}</p>
                    </div>
                </header>

                {/* Konten Ayat per Ayat */}
                <div className="p-4 bg-white rounded-xl shadow-lg border border-gray-100 space-y-6">
                    <h3 className="text-lg font-bold text-teal-700 border-b pb-2">Ayat dan Terjemahan</h3>
                    
                    {surah.verses.map((verse, index) => (
                        <div key={index} className="space-y-2 border-l-4 border-amber-300/50 pl-3">
                            {/* Nomor Ayat */}
                            <p className="text-sm font-bold text-amber-600 flex items-center gap-2">
                                <span className="w-6 h-6 flex items-center justify-center rounded-full bg-amber-100 text-xs">{index + 1}</span>
                                Ayat
                            </p>
                            
                            {/* Teks Arab (Ayat) */}
                            <p 
                                className="text-2xl sm:text-3xl font-serif text-right leading-relaxed text-gray-900 pr-1" 
                                dir="rtl" 
                                lang="ar"
                                style={{ wordBreak: 'break-word' }}
                            >
                                {verse.ar}
                            </p>
                            
                            {/* Terjemahan (Arti) */}
                            <p className="text-sm italic text-gray-600 border-t border-gray-100 pt-2">
                                {verse.id}
                            </p>
                        </div>
                    ))}
                </div>

            </div>
        );
    };

    // Halaman Daftar Surah
    const SurahListView = ({ onSelectSurah }) => (
        <div className="space-y-6 pb-20">
            <header>
                <h1 className="text-2xl font-bold text-gray-800">Al-Qur'an (Juz 30)</h1>
                <p className="text-sm text-gray-500">Daftar {SHORT_SURAHS.length} surah pendek (Juz Amma)</p>
            </header>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                {SHORT_SURAHS.map((surah) => (
                    <div 
                        key={surah.id} 
                        className="bg-white p-4 rounded-xl shadow-md hover:shadow-lg transition-shadow border border-gray-100 flex justify-between items-center group cursor-pointer active:scale-[0.98]"
                        onClick={() => onSelectSurah(surah)} // Trigger perpindahan ke halaman detail
                    >
                        {/* Kiri: Nomor, Nama, Arti */}
                        <div className="flex items-center gap-4 flex-1 min-w-0">
                            <div className="w-10 h-10 bg-teal-50 rounded-full flex-shrink-0 flex items-center justify-center text-teal-700 font-bold text-sm relative">
                                <span className="absolute rotate-45 w-8 h-8 border-2 border-teal-200 rounded-sm"></span>
                                <span className="relative">{surah.id}</span>
                            </div>
                            <div className="min-w-0">
                                <h4 className="font-bold text-gray-800 group-hover:text-teal-600 transition-colors truncate">{surah.name}</h4>
                                <p className="text-xs text-gray-500 truncate">{surah.translation} • {surah.ayat} Ayat</p>
                            </div>
                        </div>
                        {/* Kanan: Arab & Chevron */}
                        <div className="text-right flex flex-col items-end flex-shrink-0 ml-2 max-w-[50%]">
                             <p className="font-serif text-lg text-gray-700 mb-1 truncate w-full whitespace-nowrap overflow-hidden text-ellipsis" dir="rtl">
                                {surah.arabic}
                             </p>
                             <ChevronRight className="w-5 h-5 text-gray-300 group-hover:text-teal-500" />
                        </div>
                    </div>
                ))}
            </div>
        </div>
    );
    
    // View utama untuk Tab 'Quran'
    const QuranView = () => {
        if (quranView.page === 'detail' && quranView.selectedSurah) {
            return (
                <SurahDetailView 
                    surah={quranView.selectedSurah} 
                    onBack={() => setQuranView({ page: 'list', selectedSurah: null })} 
                />
            );
        }

        return (
            <SurahListView 
                onSelectSurah={(surah) => setQuranView({ page: 'detail', selectedSurah: surah })} 
            />
        );
    };


    const HomeView = () => (
        <div className="space-y-6 pb-20">
            {/* Header */}
            <header className="flex justify-between items-start">
                <div>
                    <h1 className="text-2xl font-bold text-gray-800">Assalamualaikum, <span className="text-teal-600">{userName}</span></h1>
                    <p className="text-sm text-gray-500 mt-1 flex items-center gap-1">
                        <LocationIcon className="w-4 h-4" /> {location}
                    </p>
                </div>
                <div className="text-right hidden sm:block">
                    <p className="text-sm font-bold text-gray-700">{dateGregorian}</p>
                    <p className="text-xs text-teal-600 font-medium">{dateHijri}</p>
                </div>
            </header>

            <ActivePrayerCard prayerData={activePrayerData} haidStatus={haidStatus} />
            
            <DailyDhikrCard dhikrData={todaysDhikr} />

            <div>
                <h3 className="text-lg font-bold text-gray-800 mb-3 flex items-center">
                    Jadwal Salat Hari Ini
                </h3>
                <div className="space-y-3">
                    {prayerTimes.map(prayer => (
                        <PrayerListItem key={prayer.key} prayer={prayer} />
                    ))}
                </div>
            </div>
        </div>
    );

    const QiblaView = () => {
        // Malang Coordinates
        const userLat = -7.9666;
        const userLon = 112.6326;
        const qiblaAngle = calculateQiblaAngle(userLat, userLon);

        return (
            <div className="space-y-8 pb-20 flex flex-col items-center justify-center min-h-[60vh]">
                <header className="text-center">
                    <h1 className="text-2xl font-bold text-gray-800">Arah Kiblat</h1>
                    <p className="text-sm text-gray-500">Arah Ka'bah dari {location}</p>
                </header>

                <div className="relative w-64 h-64">
                    {/* Compass Ring */}
                    <div className="absolute inset-0 border-4 border-gray-200 rounded-full flex items-center justify-center shadow-inner bg-white">
                        <div className="absolute top-2 text-xs font-bold text-gray-400">U</div>
                        <div className="absolute bottom-2 text-xs font-bold text-gray-400">S</div>
                        <div className="absolute left-2 text-xs font-bold text-gray-400">B</div>
                        <div className="absolute right-2 text-xs font-bold text-gray-400">T</div>
                    </div>

                    {/* Qibla Indicator (Rotated) */}
                    <div 
                        className="absolute inset-0 flex items-center justify-center transition-transform duration-1000 ease-out"
                        style={{ transform: `rotate(${qiblaAngle}deg)` }}
                    >
                        <div className="flex flex-col items-center -mt-32">
                            <div className="w-8 h-8 bg-teal-600 rotate-45 rounded-tl-full rounded-br-full shadow-lg mb-1 z-10"></div>
                            <div className="h-32 w-1 bg-gradient-to-b from-teal-500 to-transparent opacity-50"></div>
                            <div className="w-3 h-3 bg-teal-700 rounded-full absolute top-[50%] mt-[-6px]"></div>
                        </div>
                    </div>
                </div>

                <div className="bg-teal-50 px-6 py-4 rounded-xl text-center">
                    <p className="text-sm text-teal-800 font-medium">Sudut Kiblat</p>
                    <p className="text-3xl font-bold text-teal-700">{qiblaAngle.toFixed(2)}°</p>
                    <p className="text-xs text-teal-600">dari Utara searah jarum jam</p>
                </div>
            </div>
        );
    };

    const SettingsView = () => {
        const [tempStartDate, setTempStartDate] = useState(haidData.startDate || '');
        const [tempEndDate, setTempEndDate] = useState(haidData.endDate || '');

        const handleSaveHaid = () => {
            if (tempStartDate && tempEndDate) {
                saveHaidData({ startDate: tempStartDate, endDate: tempEndDate });
            }
        };

        return (
            <div className="space-y-6 pb-20">
                <header>
                    {/* Judul diubah menjadi Moon */}
                    <h1 className="text-2xl font-bold text-gray-800">Moon</h1>
                    <p className="text-sm text-gray-500">Pelacak Haid & Pengaturan</p>
                </header>

                {/* Profile Section */}
                <div className="bg-white p-6 rounded-xl shadow-sm border border-gray-100 flex items-center gap-4">
                    <div className="w-16 h-16 bg-gradient-to-br from-teal-400 to-emerald-600 rounded-full flex items-center justify-center text-white text-xl font-bold">
                        {userName.charAt(0)}
                    </div>
                    <div>
                        <h3 className="font-bold text-gray-800 text-lg">{userName}</h3>
                        <p className="text-sm text-gray-500">{location}</p>
                    </div>
                </div>

                {/* Haid Tracker Section */}
                <div className="bg-white rounded-xl shadow-md overflow-hidden border border-pink-100">
                    <div className="bg-pink-50 p-4 border-b border-pink-100 flex items-center gap-2">
                        <MoonIcon className="w-5 h-5 text-pink-600" />
                        <h3 className="font-bold text-pink-700">Pelacak Haid (Moon Mode)</h3>
                    </div>
                    <div className="p-6 space-y-4">
                        <p className="text-sm text-gray-600">
                            Aktifkan mode ini untuk menangguhkan pengingat salat selama masa haid.
                        </p>
                        
                        {haidStatus.isHaid && (
                            <div className="bg-pink-100 text-pink-800 p-3 rounded-lg text-sm font-medium mb-4 text-center">
                                Sedang berlangsung: Hari ke-{haidStatus.haidDay} dari {haidStatus.totalDays}
                            </div>
                        )}

                        <div className="grid grid-cols-2 gap-4">
                            <div>
                                <label className="block text-xs font-bold text-gray-500 mb-1">Tanggal Mulai</label>
                                <input 
                                    type="date" 
                                    value={tempStartDate}
                                    onChange={(e) => setTempStartDate(e.target.value)}
                                    className="w-full p-2 border border-gray-200 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-pink-300"
                                />
                            </div>
                            <div>
                                <label className="block text-xs font-bold text-gray-500 mb-1">Tanggal Selesai</label>
                                <input 
                                    type="date" 
                                    value={tempEndDate}
                                    onChange={(e) => setTempEndDate(e.target.value)}
                                    className="w-full p-2 border border-gray-200 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-pink-300"
                                />
                            </div>
                        </div>

                        <button 
                            onClick={handleSaveHaid}
                            className="w-full bg-pink-600 text-white font-bold py-2 rounded-lg hover:bg-pink-700 transition-colors active:scale-95"
                        >
                            Simpan Jadwal
                        </button>
                    </div>
                </div>
            </div>
        );
    };

    if (isLoading) {
        return (
            <div className="min-h-screen bg-gray-50 flex flex-col items-center justify-center p-4">
                <div className="w-12 h-12 border-4 border-teal-200 border-t-teal-600 rounded-full animate-spin mb-4"></div>
                <p className="text-gray-500 font-medium">Memuat data...</p>
            </div>
        );
    }

    // Tentukan apakah navigasi harus ditampilkan (sembunyikan saat di halaman detail Quran)
    const showNav = activeTab !== 'Quran' || quranView.page === 'list';


    return (
        <div className="min-h-screen bg-gray-50 font-sans text-gray-900 max-w-md mx-auto shadow-2xl relative overflow-hidden">
            {/* Main Content Area */}
            <main className="h-full overflow-y-auto p-4 md:p-6 no-scrollbar" style={{ paddingBottom: showNav ? '5rem' : '1rem' }}>
                {activeTab === 'Home' && <HomeView />}
                {activeTab === 'Quran' && <QuranView />}
                {activeTab === 'Qibla' && <QiblaView />}
                {activeTab === 'Moon' && <SettingsView />}
            </main>

            {/* Bottom Navigation (Hanya tampil jika bukan di halaman detail Quran) */}
            {showNav && (
                <nav className="fixed bottom-0 w-full max-w-md bg-white border-t border-gray-200 pb-safe pt-1 px-4 flex justify-between items-center z-50">
                    <NavItem isActive={activeTab === 'Home'} label="Beranda" onClick={() => setActiveTab('Home')}>
                        <HomeIcon />
                    </NavItem>
                    <NavItem isActive={activeTab === 'Quran'} label="Al-Qur'an" onClick={() => { setActiveTab('Quran'); setQuranView({ page: 'list', selectedSurah: null }); }}>
                        <QuranIcon />
                    </NavItem>
                    <NavItem isActive={activeTab === 'Qibla'} label="Kiblat" onClick={() => setActiveTab('Qibla')}>
                        <CompassIcon />
                    </NavItem>
                    <NavItem isActive={activeTab === 'Moon'} label="Moon" onClick={() => setActiveTab('Moon')}>
                        <MoonIcon />
                    </NavItem>
                </nav>
            )}
        </div>
    );
}
