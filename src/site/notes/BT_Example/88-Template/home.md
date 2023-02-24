```dataviewjs
>
let setting = {};
let history = Object.assign(JSON.parse(await app.vault.adapter.read(".obsidian/.diary-stats")));
let today = moment().format("YYYY-MM-DD");
let moonIndex = moment().diff(moment().startOf('year'),"hours");
if (history.hasOwnProperty(today))
{let weather=history[today].weather;
let todayweather = weather[0];
setting.iconDay = weather[0].iconDay;
setting.windSpeedDay = weather[0].windSpeedDay;
setting.windSpeedNight = weather[0].windSpeedNight;
await dv.view("dv_weatherSvg",setting)
let desc = ` <%+ tp.date.now("A好，今天是YYYY年MM月Do dddd") %> ，${todayweather.city} ${todayweather.textDay}， ${todayweather.tempMin}~${todayweather.tempMax}℃  ${todayweather.air} ${todayweather.windydesc} [[最近天气查询|]] 云朵充盈了${todayweather.cloud}%的天空\n顺便，月亮会在${todayweather.moonrise} 时浮起，${todayweather.moonset} 时沉落\n 如果足够幸运碰见它的话，我想它应该是这样的`;
dv.paragraph(desc + `<img style="margin-top:-50px;vertical-align: bottom; -webkit-clip-path: circle(42.55% at 50% 50%);" width="50" alt="|inl" src="https://svs.gsfc.nasa.gov/vis/a000000/a004900/a004955/frames/216x216_1x1_30p/moon.${moonIndex}.jpg">`);
}
>
>```