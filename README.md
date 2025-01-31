# nymey-images-anime
图片自动下载nymey
const fetch = (url, init) => import('node-fetch').then(module => module.default(url, init));
const path = require('path');
const fs = require('fs');
const https = require('https');

const updateContentJson = async () => {
    try {
        // 请求原网站获取HTML
        const htmlResponse = await fetch('https://nymey.com/geo-restriction/', {
            agent: new https.Agent({
                rejectUnauthorized: false
            })
        });
        const html = await htmlResponse.text();
        const tokenMatch = html.match(/localStorage.setItem\('access_token', "([^"]+)/);
        const accessToken = tokenMatch ? tokenMatch[1] : null;

        if (!accessToken) {
            console.error('Failed to retrieve access token');
            return;
        }

        // 使用获取到的access token请求内容数据
        const dataResponse = await fetch('https://apigateway.muvi.com/content', {
            method: 'POST',
            headers: {
                'accept': 'application/json, text/plain, */*',
                'authorization': `Bearer ${accessToken}`,
                'cache-control': 'no-cache',
                'content-type': 'application/json',
                'origin': 'https://nymey.com',
                'referer': 'https://nymey.com/',
                'sec-ch-ua': '"Not-A.Brand";v="99", "Chromium";v="124"',
                'sec-ch-ua-mobile': '?1',
                'sec-ch-ua-platform': 'Android',
                'sec-fetch-dest': 'empty',
                'sec-fetch-mode': 'cors',
                'sec-fetch-site': 'cross-site',
                'user-agent': 'Mozilla/5.0 (Linux; Android 10; K) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Mobile Safari/537.36'
            },
            body: JSON.stringify({
                query: `{featuredContentList(user_type:":user_type",end_user_uuid: ":me",ip_address:":ip",app_token: ":app_token", product_key: ":product_key", store_key:":store_key", user_uuid:":me", feature_section_uuid:"",page_content:1,per_page_content:12,content_asset_type:"", maturity_rating_uuid: ":maturity_rating_uuid", profile_uuid:":profile_uuid"){featured_content_list {section_content_list {content_list {posters {website {file_url} mobile_apps {file_url}} banners {website {file_url} mobile_apps {file_url}} trailer_details {file_url}}}}}}`
            }),
            agent: new https.Agent({
                rejectUnauthorized: false
            })
        });

        if (!dataResponse.ok) {
            console.error('Failed to fetch data from API', dataResponse.statusText);
            return;
        }

        const newData = await dataResponse.json();

        // 仅提取需要的file_url值和计算文件大小
        const getFileSize = async (url) => {
            try {
                const res = await fetch(url, { method: 'HEAD' });
                const size = res.headers.get('content-length');
                return size ? (size / (1024 * 1024)).toFixed(2) : '0';
            } catch (error) {
                console.error('Error fetching file size:', error);
                return '0';
            }
        };

        const extractedData = [];
        const seenUrls = new Set();

        for (const section of newData.data.featuredContentList.featured_content_list) {
            for (const content of section.section_content_list.content_list) {
                const urls = [
                    content.posters?.website?.[0]?.file_url,
                    content.posters?.mobile_apps?.[0]?.file_url,
                    content.banners?.website?.[0]?.file_url,
                    content.banners?.mobile_apps?.[0]?.file_url
                ];

                for (const url of urls) {
                    if (url && !seenUrls.has(url)) {
                        seenUrls.add(url);
                        const size = await getFileSize(url);
                        extractedData.push({ url, size });
                    }
                }

                const trailerUrl = content.trailer_details?.file_url;
                if (trailerUrl && !seenUrls.has(trailerUrl)) {
                    seenUrls.add(trailerUrl);
                    const size = await getFileSize(trailerUrl);
                    extractedData.push({ url: trailerUrl, size });
                }
            }
        }

        // 读取现有的 content.json 文件
        const existingDataPath = path.join(__dirname, 'content.json');
        let existingData = [];
        if (fs.existsSync(existingDataPath)) {
            const existingDataContent = fs.readFileSync(existingDataPath, 'utf-8');
            existingData = JSON.parse(existingDataContent);
        }

        // 将新数据添加到现有数据前面
        const updatedData = [...extractedData, ...existingData];

        // 写入更新后的数据到 content.json 文件
        fs.writeFileSync(existingDataPath, JSON.stringify(updatedData, null, 2));
        console.log('content.json has been updated.');
    } catch (error) {
        console.error('Error fetching data:', error);
    }
};

// 手动运行一次更新 content.json
updateContentJson();


#


<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Image Gallery</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f0f0f0;
            display: flex;
            flex-direction: column;
            align-items: center;
            margin: 0;
            padding: 20px;
        }
        h1 {
            color: #333;
        }
        .gallery {
            display: flex;
            flex-wrap: wrap;
            gap: 20px;
            justify-content: center;
        }
        .card {
            background-color: #fff;
            border: 1px solid #ddd;
            border-radius: 10px;
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
            overflow: hidden;
            width: 300px;
            text-align: center;
            transition: transform 0.2s;
        }
        .card img {
            width: 100%;
            height: auto;
        }
        .card .content {
            padding: 10px;
        }
        .card .content a {
            display: block;
            color: #007BFF;
            text-decoration: none;
            margin-top: 10px;
            word-wrap: break-word;
        }
        .card .content a:hover {
            text-decoration: underline;
        }
        .card .size {
            margin-top: 5px;
            font-size: 0.9em;
            color: #555;
        }
        .trailer-urls {
            margin-top: 20px;
            width: 80%;
            max-width: 800px;
        }
        .trailer-urls div {
            background-color: #fff;
            border: 1px solid #ddd;
            border-radius: 5px;
            padding: 10px;
            margin: 5px 0;
            box-shadow: 0 0 5px rgba(0, 0, 0, 0.1);
            word-wrap: break-word;
        }
        .toggle-button {
            cursor: pointer;
            margin: 10px;
            display: flex;
            align-items: center;
            justify-content: center;
            width: 50px;
            height: 50px;
            border-radius: 50%;
            background: linear-gradient(135deg, #6e8efb, #a777e3);
            box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
            transition: transform 0.3s;
        }
        .toggle-button:hover {
            transform: scale(1.1);
        }
        .toggle-button svg {
            fill: #fff;
        }
        .small-card {
            width: 150px;
        }
        .large-card {
            width: 300px;
        }
        .filter-dropdown {
            margin-bottom: 20px;
        }
    </style>
</head>
<body>
    <h1>Image Gallery</h1>
    <div class="filter-dropdown">
        <label for="size-filter">Filter by size:</label>
        <select id="size-filter">
            <option value="all">All</option>
            <option value="5">Greater than 5MB</option>
            <option value="10">Greater than 10MB</option>
        </select>
    </div>
    <div class="toggle-button" id="toggle-button">
        <svg width="24" height="24" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg">
            <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zM12 19c-3.87 0-7-3.13-7-7s3.13-7 7-7 7 3.13 7 7-3.13 7-7 7zm-1-4h2v2h-2zm0-10h2v6h-2z" fill="#fff"/>
        </svg>
    </div>
    <div class="gallery" id="gallery"></div>
    <div class="trailer-urls" id="trailer-urls"></div>

    <script>
        async function fetchData() {
            try {
                const response = await fetch('./content.json');
                const data = await response.json();
                
                displayImages(data);
                displayTrailerUrls(data);
            } catch (error) {
                console.error('Error fetching data:', error);
            }
        }

        function displayImages(data) {
            const gallery = document.getElementById('gallery');
            gallery.innerHTML = ''; // Clear existing content
            const filterValue = document.getElementById('size-filter').value;

            data.forEach(content => {
                const { url, size, isTrailer } = content;
                if (isTrailer) return; // Skip trailers for the image gallery

                const sizeMB = parseFloat(size);
                if (filterValue === '5' && sizeMB <= 5) return;
                if (filterValue === '10' && sizeMB <= 10) return;

                const cardElement = document.createElement('div');
                cardElement.classList.add('card', 'large-card');

                const imgElement = document.createElement('img');
                imgElement.src = url;

                const contentElement = document.createElement('div');
                contentElement.classList.add('content');
                contentElement.innerHTML = `<a href="${url}" target="_blank">View Image</a>`;

                const sizeElement = document.createElement('div');
                sizeElement.classList.add('size');
                sizeElement.textContent = `${size} MB`;

                cardElement.appendChild(imgElement);
                cardElement.appendChild(contentElement);
                cardElement.appendChild(sizeElement);
                gallery.appendChild(cardElement);
            });
        }

        function displayTrailerUrls(data) {
            const trailerUrlsContainer = document.getElementById('trailer-urls');
            trailerUrlsContainer.innerHTML = '<h2>Trailer URLs</h2>';
            data.forEach(content => {
                const { url, size, isTrailer } = content;
                if (isTrailer) {
                    const urlElement = document.createElement('div');
                    urlElement.textContent = `${url} (${size} MB)`;
                    trailerUrlsContainer.appendChild(urlElement);
                }
            });
        }

        document.getElementById('toggle-button').addEventListener('click', () => {
            const gallery = document.getElementById('gallery');
            gallery.classList.toggle('wrap');
            const cards = document.querySelectorAll('.card');
            cards.forEach(card => {
                card.classList.toggle('small-card');
                card.classList.toggle('large-card');
            });
        });

        document.getElementById('size-filter').addEventListener('change', () => {
            fetchData();
        });

        fetchData();
    </script>
</body>
</html>
