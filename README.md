<!DOCTYPE html>
<html lang="mn">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>SkySobo</title>

<!-- Favicon -->
<link rel="icon" type="image/png" href="https://upload.wikimedia.org/wikipedia/commons/thumb/e/e1/Cloud_icon.svg/1024px-Cloud_icon.svg.png" />

<style>
  body {
    margin:0; padding:0;
    font-family: Arial, sans-serif;
    display: flex; justify-content: center; align-items: center;
    height: 100vh;
    overflow: hidden;
    transition: background 1s;
    background: linear-gradient(to right, #74ebd5, #ACB6E5); /* default өдөр өнгө */
  }
  .container {
    background: rgba(255,255,255,0.95);
    padding: 30px 40px;
    border-radius: 20px;
    text-align: center;
    width: 360px;
    position: relative;
    z-index:2;
  }
  input { width: 90%; padding: 10px; margin-bottom: 10px; border-radius:5px; border:1px solid #ccc; }
  button { padding: 10px 20px; border-radius:5px; border:none; background-color:#4CAF50; color:white; cursor:pointer; }
  button:hover{background-color:#45a049;}
  .icon { width:80px; height:80px; }
  .temperature { font-size:32px; margin:10px 0; }
  .description { font-size:20px; text-transform:capitalize; margin-bottom:10px; }

  /* Бороо */
  .rain { position:absolute; top:0; left:0; width:100%; height:100%; pointer-events:none; z-index:1; }
  .drop { position:absolute; width:2px; height:10px; background:rgba(255,255,255,0.7); animation: fall linear infinite; }
  @keyframes fall { 0%{transform:translateY(-10px);} 100%{transform:translateY(100vh);} }

  /* Үүл */
  .cloud {
    position:absolute;
    width:100px; height:60px;
    background:rgba(255,255,255,0.8);
    border-radius:50%;
    top:50px;
    animation: cloudMove 80s linear infinite; /* удаан хөдөлгөөн */
  }
  .cloud:before, .cloud:after {
    content:""; position:absolute;
    background:rgba(255,255,255,0.8); width:60px; height:60px; border-radius:50%;
  }
  .cloud:before { left:-30px; top:0; }
  .cloud:after { right:-30px; top:0; }
  @keyframes cloudMove { 0%{left:-150px;} 100%{left:100%;} }

  /* Нар */
  .sun { position:absolute; width:80px; height:80px; border-radius:50%; background:yellow; top:20px; right:20px; box-shadow: 0 0 50px yellow; animation: sunRotate 30s linear infinite; }
  @keyframes sunRotate { 0%{transform:rotate(0deg);} 100%{transform:rotate(360deg);} }

  /* Autocomplete list */
  .autocomplete-items { position: absolute; border: 1px solid #d4d4d4; border-bottom:none; border-top:none; z-index:99; top:100%; left:0; right:0; background:white; max-height:200px; overflow-y:auto; }
  .autocomplete-items div { padding: 10px; cursor:pointer; background-color:#fff; border-bottom:1px solid #d4d4d4; }
  .autocomplete-items div:hover { background-color:#e9e9e9; }
</style>
</head>
<body>

<div class="container">
  <h1>SkySobo</h1>
  <div style="position:relative;">
    <input type="text" id="cityInput" placeholder="Хот/сумын нэрээ бичнэ үү" autocomplete="off">
    <div id="autocomplete-list" class="autocomplete-items"></div>
  </div>
  <button onclick="getWeather()">Хайх</button>
  <div id="weatherInfo" style="display:none;">
    <h2 id="cityName"></h2>
    <img id="icon" class="icon" src="" alt="icon">
    <p class="temperature" id="temperature"></p>
    <p class="description" id="description"></p>
  </div>
</div>

<div class="rain" id="rainContainer"></div>
<div class="cloud" id="cloud1" style="display:none;"></div>
<div class="cloud" id="cloud2" style="display:none; top:120px;"></div>
<div class="sun" id="sun" style="display:none;"></div>

<script>
// Монголын бүх аймаг болон дүүргүүд
const mongoliaCities = [
  "Улаанбаатар", "Баянзүрх", "Сонгинохайрхан", "Сүхбаатар", "Чингэлтэй", "Хан-Уул", "Баянгол", "Налайх",
  "Архангай", "Баян-Өлгий", "Баянхонгор", "Булган", "Говь-Алтай", "Дорноговь", "Дорнод", "Дундговь",
  "Завхан", "Өвөрхангай", "Өмнөговь", "Сэлэнгэ", "Сүхбаатар аймаг", "Төв", "Увс", "Ховд", "Хөвсгөл",
  "Орхон", "Дархан-Уул"
];

// Autocomplete
const input = document.getElementById('cityInput');
const listContainer = document.getElementById('autocomplete-list');

input.addEventListener("input", function(){
  const val = this.value.toLowerCase();
  listContainer.innerHTML = '';
  if(!val) return false;
  mongoliaCities.forEach(city => {
    if(city.toLowerCase().includes(val)){
      const item = document.createElement('div');
      item.innerHTML = city;
      item.addEventListener('click', function(){
        input.value = city;
        listContainer.innerHTML = '';
      });
      listContainer.appendChild(item);
    }
  });
});

function createRain(){
  const container = document.getElementById('rainContainer');
  container.innerHTML='';
  for(let i=0;i<50;i++){
    const drop = document.createElement('div');
    drop.className='drop';
    drop.style.left=Math.random()*100+'%';
    drop.style.height=(5+Math.random()*10)+'px';
    drop.style.width=(1+Math.random()*2)+'px';
    drop.style.opacity=(0.3+Math.random()*0.7);
    drop.style.animationDuration=(0.5+Math.random())+'s';
    drop.style.animationDelay=Math.random()+'s';
    container.appendChild(drop);
  }
}

async function getWeather(){
  const city = document.getElementById('cityInput').value;
  const apiKey = "b6276a9379241de1a027b73b413b1500"; 
  const url=`https://api.openweathermap.org/data/2.5/weather?q=${city}&appid=${apiKey}&units=metric&lang=mn`;

  try{
    const response=await fetch(url);
    const data=await response.json();
    if(data.cod===200){
      document.getElementById('weatherInfo').style.display='block';
      document.getElementById('cityName').innerText = data.name + ', ' + data.sys.country;
      document.getElementById('temperature').innerText = Math.round(data.main.temp)+'°C';
      document.getElementById('description').innerText = data.weather[0].description;
      document.getElementById('icon').src=`https://openweathermap.org/img/wn/${data.weather[0].icon}@2x.png`;

      // Өдөр/шөнө өнгө
      const icon = data.weather[0].icon;
      const hours = new Date().getHours();
      if(icon.endsWith('d') || (hours>=6 && hours<18)){
        document.body.style.background='linear-gradient(to right, #74ebd5, #ACB6E5)';
      } else {
        document.body.style.background='linear-gradient(to right, #2c3e50, #4ca1af)';
      }

      const weatherMain = data.weather[0].main.toLowerCase();
      if(weatherMain.includes('rain')){
        createRain();
        cloud1.style.display='block';
        cloud2.style.display='block';
        sun.style.display='none';
      } else if(weatherMain.includes('cloud')){
        cloud1.style.display='block';
        cloud2.style.display='block';
        rainContainer.innerHTML='';
        sun.style.display='none';
      } else if(weatherMain.includes('clear')){
        sun.style.display='block';
        cloud1.style.display='none';
        cloud2.style.display='none';
        rainContainer.innerHTML='';
      } else {
        sun.style.display='none';
        cloud1.style.display='none';
        cloud2.style.display='none';
        rainContainer.innerHTML='';
      }

    } else{
      alert("Хот/сум олдсонгүй!");
      weatherInfo.style.display='none';
      rainContainer.innerHTML='';
      cloud1.style.display='none';
      cloud2.style.display='none';
      sun.style.display='none';
    }
  }catch(e){console.error(e);}
}
</script>

</body>
</html>
