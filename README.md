最近在学习nodejs爬虫技术，学了request模块，所以想着写一个自己的爬虫项目，研究了半天，最后选定indeed作为目标网站，通过爬取indeed的职位数据，然后开发一个自己的职位搜索引擎，目前已经上线了，虽然功能还是比较简单，但还是贴一下网址[job search engine](https://ideras.com)，证明一下这个爬虫项目是有用的。下面就来讲讲整个爬虫的思路。 

## 确定入口页面

众所周知，爬虫是需要入口页面的，通过入口页面，不断的爬取链接，最后爬取完整个网站。在这个第一步的时候，就遇到了困难，一般来说都是选取首页和列表页作为入口页面的，但是indeed的列表页面做了限制，不能爬取完整的列表，顶多只能抓取前100页，但是这没有难倒我，我发现indeed有一个**Browse Jobs** 页面，通过这个页面，可以获取indeed按地区搜索和按类型搜索的所有列表。下面贴一下这个页面的解析代码。

```
start: async (page) => {
  const host = URL.parse(page.url).hostname;
  const tasks = [];
  try {
    const $ = cheerio.load(iconv.decode(page.con, 'utf-8'), { decodeEntities: false });
    $('#states > tbody > tr > td > a').each((i, ele) => {
      const url = URL.resolve(page.url, $(ele).attr('href'));
      tasks.push({ _id: md5(url), type: 'city', host, url, done: 0, name: $(ele).text() });
    });
    $('#categories > tbody > tr > td > a').each((i, ele) => {
      const url = URL.resolve(page.url, $(ele).attr('href'));
      tasks.push({ _id: md5(url), type: 'category', host, url, done: 0, name: $(ele).text() });
    });
    const res = await global.com.task.insertMany(tasks, { ordered: false }).catch(() => {});
    res && console.log(`${host}-start insert ${res.insertedCount} from ${tasks.length} tasks`);
    return 1;
  } catch (err) {
    console.error(`${host}-start parse ${page.url} ${err}`);
    return 0;
  }
}
```

通过cheerio解析html内容，把按地区搜索和按类型搜索链接插入到数据库中。

## 爬虫架构

这里简单讲一下我的爬虫架构思路，数据库选用mongodb。每一个待爬取的页面存一条记录page，包含id,url,done,type,host等字段，id用`md5(url)`生成，避免重复。每一个type有一个对应的html内容解析方法，主要的业务逻辑都集中在这些解析方法里面，上面贴出来的代码就是例子。

爬取html采用request模块，进行了简单的封装，把callback封装成promise，方便使用async和await方式调用，代码如下。

```
const req = require('request');

const request = req.defaults({
  headers: {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/78.0.3904.108 Safari/537.36'
  },
  timeout: 30000,
  encoding: null
});

const fetch = (url) => new Promise((resolve) => {
    console.log(`down ${url} started`);
    request(encodeURI(url), (err, res, body) => {
      if (res && res.statusCode === 200) {
        console.log(`down ${url} 200`);
        resolve(body);
      } else {
        console.error(`down ${url} ${res && res.statusCode} ${err}`);
        if (res && res.statusCode) {
          resolve(res.statusCode);
        } else {
          // ESOCKETTIMEOUT 超时错误返回600
          resolve(600);
        }
      }
    });
  });
```

做了简单的反反爬处理，把user-agent改成电脑通用的user-agent，设置了超时时间30秒，其中`encoding: null`设置request直接返回buffer，而不是解析后的内容，这样的好处是如果页面是gbk或者utf-8编码，只要解析html的时候指定编码就行了，如果这里指定`encoding: utf-8`，则当页面编码是gbk的时候，页面内容会乱码。

request默认是回调函数形式，通过promise封装，如果成功，则返回页面内容的buffer，如果失败，则返回错误状态码，如果超时，则返回600，这些懂nodejs的应该很好理解。

## 完整的解析代码

```
const URL = require('url');
const md5 = require('md5');
const cheerio = require('cheerio');
const iconv = require('iconv-lite');

const json = (data) => {
  let res;
  try {
    res = JSON.parse(data);
  } catch (err) {
    console.error(err);
   }
  return res;
};

const rules = [
  /\/jobs\?q=.*&sort=date&start=\d+/,
  /\/jobs\?q=&l=.*&sort=date&start=\d+/
];

const fns = {

  start: async (page) => {
    const host = URL.parse(page.url).hostname;
    const tasks = [];
    try {
      const $ = cheerio.load(iconv.decode(page.con, 'utf-8'), { decodeEntities: false });
      $('#states > tbody > tr > td > a').each((i, ele) => {
        const url = URL.resolve(page.url, $(ele).attr('href'));
        tasks.push({ _id: md5(url), type: 'city', host, url, done: 0, name: $(ele).text() });
      });
      $('#categories > tbody > tr > td > a').each((i, ele) => {
        const url = URL.resolve(page.url, $(ele).attr('href'));
        tasks.push({ _id: md5(url), type: 'category', host, url, done: 0, name: $(ele).text() });
      });
      const res = await global.com.task.insertMany(tasks, { ordered: false }).catch(() => {});
      res && console.log(`${host}-start insert ${res.insertedCount} from ${tasks.length} tasks`);
      return 1;
    } catch (err) {
      console.error(`${host}-start parse ${page.url} ${err}`);
      return 0;
    }
  },

  city: async (page) => {
    const host = URL.parse(page.url).hostname;
    const tasks = [];
    const cities = [];
    try {
      const $ = cheerio.load(iconv.decode(page.con, 'utf-8'), { decodeEntities: false });
      $('#cities > tbody > tr > td > p.city > a').each((i, ele) => {
        // https://www.indeed.com/l-Charlotte,-NC-jobs.html
        let tmp = $(ele).attr('href').match(/l-(?<loc>.*)-jobs.html/u);
        if (!tmp) {
          tmp = $(ele).attr('href').match(/l=(?<loc>.*)/u);
        }
        const { loc } = tmp.groups;
        const url = `https://www.indeed.com/jobs?l=${decodeURIComponent(loc)}&sort=date`;
        tasks.push({ _id: md5(url), type: 'search', host, url, done: 0 });
        cities.push({ _id: `${$(ele).text()}_${page.name}`, parent: page.name, name: $(ele).text(), url });
      });
      let res = await global.com.city.insertMany(cities, { ordered: false }).catch(() => {});
      res && console.log(`${host}-city insert ${res.insertedCount} from ${cities.length} cities`);

      res = await global.com.task.insertMany(tasks, { ordered: false }).catch(() => {});
      res && console.log(`${host}-city insert ${res.insertedCount} from ${tasks.length} tasks`);
      return 1;
    } catch (err) {
      console.error(`${host}-city parse ${page.url} ${err}`);
      return 0;
    }
  },

  category: async (page) => {
    const host = URL.parse(page.url).hostname;
    const tasks = [];
    const categories = [];
    try {
      const $ = cheerio.load(iconv.decode(page.con, 'utf-8'), { decodeEntities: false });
      $('#titles > tbody > tr > td > p.job > a').each((i, ele) => {
        const { query } = $(ele).attr('href').match(/q-(?<query>.*)-jobs.html/u).groups;
        const url = `https://www.indeed.com/jobs?q=${decodeURIComponent(query)}&sort=date`;
        tasks.push({ _id: md5(url), type: 'search', host, url, done: 0 });
        categories.push({ _id: `${$(ele).text()}_${page.name}`, parent: page.name, name: $(ele).text(), url });
      });
      let res = await global.com.category.insertMany(categories, { ordered: false }).catch(() => {});
      res && console.log(`${host}-category insert ${res.insertedCount} from ${categories.length} categories`);

      res = await global.com.task.insertMany(tasks, { ordered: false }).catch(() => {});
      res && console.log(`${host}-category insert ${res.insertedCount} from ${tasks.length} tasks`);
      return 1;
    } catch (err) {
      console.error(`${host}-category parse ${page.url} ${err}`);
      return 0;
    }
  },

  search: async (page) => {
    const host = URL.parse(page.url).hostname;
    const tasks = [];
    const durls = [];
    try {
      const con = iconv.decode(page.con, 'utf-8');
      const $ = cheerio.load(con, { decodeEntities: false });
      const list = con.match(/jobmap\[\d+\]= {.*}/g);
      const jobmap = [];
      if (list) {
         // eslint-disable-next-line no-eval
        list.map((item) => eval(item));
      }
      for (const item of jobmap) {
        const cmplink = URL.resolve(page.url, item.cmplnk);
        const { query } = URL.parse(cmplink, true);
        let name;
        if (query.q) {
          // eslint-disable-next-line prefer-destructuring
          name = query.q.split(' #')[0].split('#')[0];
        } else {
          const tmp = cmplink.match(/q-(?<text>.*)-jobs.html/u);
          if (!tmp) {
            // eslint-disable-next-line no-continue
            continue;
          }
          const { text } = tmp.groups;
          // eslint-disable-next-line prefer-destructuring
          name = text.replace(/-/g, ' ').split(' #')[0];
        }
        const surl = `https://www.indeed.com/cmp/_cs/cmpauto?q=${name}&n=10&returnlogourls=1&returncmppageurls=1&caret=8`;
        const burl = `https://www.indeed.com/viewjob?jk=${item.jk}&from=vjs&vjs=1`;
        const durl = `https://www.indeed.com/rpc/jobdescs?jks=${item.jk}`;
        tasks.push({ _id: md5(surl), type: 'suggest', host, url: surl, done: 0 });
        tasks.push({ _id: md5(burl), type: 'brief', host, url: burl, done: 0 });
        durls.push({ _id: md5(durl), type: 'detail', host, url: durl, done: 0 });
      }
      $('a[href]').each((i, ele) => {
        const tmp = URL.resolve(page.url, $(ele).attr('href'));
        const [url] = tmp.split('#');
        const { path, hostname } = URL.parse(url);
        for (const rule of rules) {
          if (rule.test(path)) {
            if (hostname == host) {
              // tasks.push({ _id: md5(url), type: 'list', host, url: decodeURI(url), done: 0 });
            }
            break;
          }
        }
      });

      let res = await global.com.task.insertMany(tasks, { ordered: false }).catch(() => {});
      res && console.log(`${host}-search insert ${res.insertedCount} from ${tasks.length} tasks`);

      res = await global.com.task.insertMany(durls, { ordered: false }).catch(() => {});
      res && console.log(`${host}-search insert ${res.insertedCount} from ${durls.length} tasks`);

      return 1;
    } catch (err) {
      console.error(`${host}-search parse ${page.url} ${err}`);
      return 0;
    }
  },

  suggest: async (page) => {
    const host = URL.parse(page.url).hostname;
    const tasks = [];
    const companies = [];
    try {
      const con = page.con.toString('utf-8');
      const data = json(con);
      for (const item of data) {
        const id = item.overviewUrl.replace('/cmp/', '');
        const cmpurl = `https://www.indeed.com/cmp/${id}`;
        const joburl = `https://www.indeed.com/cmp/${id}/jobs?clearPrefilter=1`;
        tasks.push({ _id: md5(cmpurl), type: 'company', host, url: cmpurl, done: 0 });
        tasks.push({ _id: md5(joburl), type: 'jobs', host, url: joburl, done: 0 });
        companies.push({ _id: id, name: item.name, url: cmpurl });
      }

      let res = await global.com.company.insertMany(companies, { ordered: false }).catch(() => {});
      res && console.log(`${host}-suggest insert ${res.insertedCount} from ${companies.length} companies`);

      res = await global.com.task.insertMany(tasks, { ordered: false }).catch(() => {});
      res && console.log(`${host}-suggest insert ${res.insertedCount} from ${tasks.length} tasks`);
      return 1;
    } catch (err) {
      console.error(`${host}-suggest parse ${page.url} ${err}`);
      return 0;
    }
  },

  // list: () => {},

  jobs: async (page) => {
    const host = URL.parse(page.url).hostname;
    const tasks = [];
    const durls = [];
    try {
      const con = iconv.decode(page.con, 'utf-8');
      const tmp = con.match(/window._initialData=(?<text>.*);<\/script><script>window._sentryData/u);
      let data;
      if (tmp) {
        const { text } = tmp.groups;
        data = json(text);
        if (data.jobList && data.jobList.pagination && data.jobList.pagination.paginationLinks) {
          for (const item of data.jobList.pagination.paginationLinks) {
            // eslint-disable-next-line max-depth
            if (item.href) {
              item.href = item.href.replace(/\u002F/g, '/');
              const url = URL.resolve(page.url, decodeURI(item.href));
              tasks.push({ _id: md5(url), type: 'jobs', host, url: decodeURI(url), done: 0 });
            }
          }
        }
        if (data.jobList && data.jobList.jobs) {
          for (const job of data.jobList.jobs) {
            const burl = `https://www.indeed.com/viewjob?jk=${job.jobKey}&from=vjs&vjs=1`;
            const durl = `https://www.indeed.com/rpc/jobdescs?jks=${job.jobKey}`;
            tasks.push({ _id: md5(burl), type: 'brief', host, url: burl, done: 0 });
            durls.push({ _id: md5(durl), type: 'detail', host, url: durl, done: 0 });
          }
        }
      } else {
        console.log(`${host}-jobs ${page.url} has no _initialData`);
      }
      let res = await global.com.task.insertMany(tasks, { ordered: false }).catch(() => {});
      res && console.log(`${host}-search insert ${res.insertedCount} from ${tasks.length} tasks`);

      res = await global.com.task.insertMany(durls, { ordered: false }).catch(() => {});
      res && console.log(`${host}-search insert ${res.insertedCount} from ${durls.length} tasks`);

      return 1;
    } catch (err) {
      console.error(`${host}-jobs parse ${page.url} ${err}`);
      return 0;
    }
  },

  brief: async (page) => {
    const host = URL.parse(page.url).hostname;
    try {
      const con = page.con.toString('utf-8');
      const data = json(con);
      data.done = 0;
      data.views = 0;
      data.host = host;
      // format publish date
      if (data.vfvm && data.vfvm.jobAgeRelative) {
        const str = data.vfvm.jobAgeRelative;
        const tmp = str.split(' ');
        const [first, second] = tmp;
        if (first == 'Just' || first == 'Today') {
          data.publishDate = Date.now();
        } else {
          const num = first.replace(/\+/, '');
          if (second == 'hours') {
            const date = new Date();
            const time = date.getTime();
            // eslint-disable-next-line no-mixed-operators
            date.setTime(time - num * 60 * 60 * 1000);
            data.publishDate = date.getTime();
          } else if (second == 'days') {
            const date = new Date();
            const time = date.getTime();
            // eslint-disable-next-line no-mixed-operators
            date.setTime(time - num * 24 * 60 * 60 * 1000);
            data.publishDate = date.getTime();
          } else {
            data.publishDate = Date.now();
          }
        }
      }
      await global.com.job.updateOne({ _id: data.jobKey }, { $set: data }, { upsert: true }).catch(() => { });

      const tasks = [];
      const url = `https://www.indeed.com/jobs?l=${data.jobLocationModel.jobLocation}&sort=date`;
      tasks.push({ _id: md5(url), type: 'search', host, url, done: 0 });
      const res = await global.com.task.insertMany(tasks, { ordered: false }).catch(() => {});
      res && console.log(`${host}-brief insert ${res.insertedCount} from ${tasks.length} tasks`);
      return 1;
    } catch (err) {
      console.error(`${host}-brief parse ${page.url} ${err}`);
      return 0;
    }
  },

  detail: async (page) => {
    const host = URL.parse(page.url).hostname;
    try {
      const con = page.con.toString('utf-8');
      const data = json(con);
      const [jobKey] = Object.keys(data);
      await global.com.job.updateOne({ _id: jobKey }, { $set: { content: data[jobKey], done: 1 } }).catch(() => { });
      return 1;
    } catch (err) {
      console.error(`${host}-detail parse ${page.url} ${err}`);
      return 0;
    }
  },

  run: (page) => {
    if (page.type == 'list') {
      page.type = 'search';
    }
    const fn = fns[page.type];
    if (fn) {
      return fn(page);
    }
    console.error(`${page.url} parser not found`);
    return 0;
  }

};

module.exports = fns;
```

每一个解析方法都会插入一些新的链接，新的链接记录都会有一个type字段，通过type字段，可以知道新的链接的解析方法，这样就能完整解析所有的页面了。例如start方法会插入type为city和category的记录，type为city的页面记录的解析方法就是`city`方法，city方法里面又会插入type为search的链接，这样一直循环，直到最后的brief和detail方法分别获取职位数据的简介和详细内容。

其实爬虫最关键的就是这些html解析方法，有了这些方法，你就能获取任何想要的结构化内容了。

## 数据索引

这部分就很简单了，有了前面获取的结构化数据，按照elasticsearch，新建一个schema，然后写个程序定时把职位数据添加到es的索引里面就行了。因为职位详情的内容有点多，我就没有把content字段添加到索引里面了，因为太占内存了，服务器内存不够用了，>_<。

## DEMO

最后还是贴上网址供大家检阅，[job search engine](https://ideras.com)。
