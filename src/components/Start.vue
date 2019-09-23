<template>
  <div v-if="logined">
    <button @click="logout">Logout</button>
    <br />
    <button @click="remote">Remote: {{isRemote}}</button>
    <br />
    <h5>Request</h5>domain:
    <input type="text" v-model="listDomain" />
    <br />display_name:
    <input type="text" v-model="listDName" />
    <br />username:
    <input type="text" v-model="listUsername" />
    <br />
    <br />
    <button @click="get">GET</button>
    <h5>Filter</h5>filter regEx:
    <input type="text" v-model="filter" />
    <br />
    <div id="fix">
      <button @click="act('silence')">Silence</button>
      <button @click="act('suspend')">Suspend</button>
    </div>
    <div v-for="acct in accts" :key="acct.id" v-if="check(acct.account)">
      <input type="checkbox" :value="acct.id" v-model="checkedAccts" />
      <b>@{{acct.account.acct}}</b>
      <div class="cvo" style="padding-top:5px;">
        <div class="area-icon">
          <img draggable="false" :src="acct.account.avatar" width="40" class="prof-img" />
        </div>
        <div class="area-display_name">
          <div class="flex-name">
            <span class="user">{{acct.account.display_name}}</span>
            <span
              class="sml gray"
              style="overflow: hidden;white-space: nowrap;text-overflow: ellipsis;user-select:auto; cursor:text;"
            >@{{acct.account.acct}}</span>
          </div>
        </div>
        <div class="area-toot acct-note" v-html="acct.account.note.replace(/<br\s?\/?>.+/g, '...')"></div>
        <div style="justify-content:flex-start;top:5px;flex-wrap: wrap;" class="area-actions">
          <div class="cbadge">Following: {{acct.account.following_count}}</div>
          <div class="cbadge">Followers: {{acct.account.followers_count}}</div>
          <div class="cbadge">Last status: {{acct.account.last_status_at}}</div>
          <div class="cbadge">Created at: {{acct.account.created_at}}</div>
        </div>
      </div>
    </div>
    <button @click="more" v-if="accts">More</button>
    <br />
    <br />
  </div>
  <div v-else>
    <input type="text" v-model="domain" />
    <button @click="login" placeholder="domain.tld">Login</button>
  </div>
</template>

<script>
export default {
  name: "Start",
  data() {
    return {
      domain: null,
      logined: false,
      accts: null,
      checkedAccts: [],
      isRemote: false,
      listDomain: "",
      listDName: "",
      listUsername: "",
      filter: null,
      moreLoader: false
    };
  },
  mounted: function() {
    if (this.loggedIn()) {
      this.logined = true;
      this.get();
    } else {
      let m = location.search.match(/\?code=([a-zA-Z-0-9-_+=]+)/);
      if (!m) {
        return;
      }
      let vm = this;
      let domain = localStorage.getItem("domain");
      let start = "https://" + domain + "/oauth/token";
      let id = localStorage.getItem("client_id");
      let secret = localStorage.getItem("client_secret");
      localStorage.removeItem("client_id");
      localStorage.removeItem("client_secret");
      let httpreq = new XMLHttpRequest();
      httpreq.open("POST", start, true);
      httpreq.setRequestHeader("Content-Type", "application/json");
      httpreq.responseType = "json";
      httpreq.send(
        JSON.stringify({
          grant_type: "authorization_code",
          redirect_uri: location.origin + location.pathname,
          client_id: id,
          client_secret: secret,
          code: m[1]
        })
      );
      httpreq.onreadystatechange = function() {
        if (httpreq.readyState == 4) {
          let json = httpreq.response;
          if (json["access_token"]) {
            localStorage.setItem("token", json["access_token"]);
            localStorage.removeItem("client_id");
            localStorage.removeItem("client_secret");
            location.href = "./";
          }
        }
      };
    }
  },
  methods: {
    login: function(event) {
      const start = "https://" + this.domain + "/api/v1/apps";
      localStorage.setItem("domain", this.domain);
      const httpreq = new XMLHttpRequest();
      let vm = this;
      httpreq.open("POST", start, true);
      httpreq.setRequestHeader("Content-Type", "application/json");
      httpreq.responseType = "json";
      const red = location.origin + location.pathname;
      httpreq.send(
        JSON.stringify({
          scopes: "admin:read admin:write",
          client_name: "spamastodon",
          redirect_uris: red
        })
      );
      httpreq.onreadystatechange = function() {
        if (httpreq.readyState == 4) {
          let json = httpreq.response;
          const auth =
            "https://" +
            vm.domain +
            "/oauth/authorize?client_id=" +
            json["client_id"] +
            "&client_secret=" +
            json["client_secret"] +
            "&response_type=code&scope=admin:read+admin:write&redirect_uri=" +
            encodeURIComponent(red);
          localStorage.setItem("client_id", json["client_id"]);
          localStorage.setItem("client_secret", json["client_secret"]);
          location.href = auth;
        }
      };
    },
    loggedIn: function() {
      return !!localStorage.getItem("token");
    },
    remote: function() {
      if (this.isRemote) {
        this.isRemote = false;
      } else {
        this.isRemote = true;
      }
      this.get();
    },
    get: function() {
      if (!this.moreLoader) {
        var next = false;
        this.paging = "";
      } else {
        var next = true;
      }
      let domain = localStorage.getItem("domain");
      let at = localStorage.getItem("token");
      let start = "https://" + domain + "/api/v1/admin/accounts";
      if (this.isRemote) {
        var remoteCk = "true";
      } else {
        var remoteCk = "";
      }
      let params = {
        remote: remoteCk,
        by_domain: this.listDomain,
        display_name: this.listDName,
        username: this.listUsername,
        max_id: this.paging
      };
      let esc = encodeURIComponent;
      let query = Object.keys(params)
        .map(k => esc(k) + "=" + esc(params[k]))
        .join("&");
      const httpreq = new XMLHttpRequest();
      let vm = this;
      httpreq.open("GET", start + "?" + query, true);
      httpreq.setRequestHeader("Content-Type", "application/json");
      httpreq.setRequestHeader("Authorization", "Bearer " + at);
      httpreq.responseType = "json";
      httpreq.send();
      httpreq.onreadystatechange = function() {
        if (httpreq.readyState == 4) {
          let link = httpreq.getResponseHeader("link");
          if (link) {
            let matches = link.match(/[?&]{1}max_id=([0-9]+)/);
            if (matches) {
              if (matches.length > 1) {
                vm.paging = matches[1];
              }
            }
          }
          let json = httpreq.response;
          if (next) {
            let arr = vm.accts;
            vm.accts = arr.concat(json);
          } else {
            vm.accts = json;
          }
          console.log(vm.accts);
        }
      };
    },
    more: function() {
      this.moreLoader = true;
      this.get(true);
    },
    act: function(act) {
      if (!confirm("action: " + act)) {
        return false;
      }
      let domain = localStorage.getItem("domain");
      let at = localStorage.getItem("token");
      let targets = this.checkedAccts;
      let len = targets.length;
      let vm = this;
      var ct = 0;
      for (var i = 0; i < len; i++) {
        let httpreq = new XMLHttpRequest();
        let start =
          "https://" +
          domain +
          "/api/v1/admin/accounts/" +
          targets[i] +
          "/action";
        httpreq.open("POST", start, true);
        httpreq.setRequestHeader("Content-Type", "application/json");
        httpreq.setRequestHeader("Authorization", "Bearer " + at);
        httpreq.responseType = "json";
        httpreq.send(
          JSON.stringify({
            type: act
          })
        );
        httpreq.onreadystatechange = function() {
          if (httpreq.readyState == 4) {
            ct++;
            if (ct > len) {
              alert("complete");
            }
          }
        };
      }
    },
    check: function(acct) {
      if (acct.id < 0) {
        return false;
      }
      if (this.filter) {
        const regExp = new RegExp(this.filter, "gi");
        if (
          acct.acct.match(regExp) ||
          acct.display_name.match(regExp) ||
          acct.note.match(regExp)
        ) {
          return true;
        } else {
          return false;
        }
      } else {
        return true;
      }
    },
    logout: function() {
      if (confirm("logout?")) {
        localStorage.clearItem();
        location.reload();
      }
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#fix {
  position: fixed;
  bottom: 0;
  width: 100vw;
  z-index: 999;
  background-color: white;
}
h1,
h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
.cvo {
  user-select: text;
  padding-left: 5px;
  border-bottom: 0.5px solid;
  padding-right: 2px;
  word-break: break-word;
  width: calc(100% - 10px);
  display: grid;
  grid-template-columns: 43px 1fr 0;
  grid-template-rows: auto 1.6rem 1fr auto auto;
  grid-template-areas: "notice notice notice" "icon display_name display_name" "space toot toot" "space additional additional" "vis actions side";
}
.area-icon {
  width: 40px;
  margin: 2px;
  grid-area: icon;
}

.area-display_name {
  user-select: text;
  height: 1.5em;
  margin: 2px;
  margin-left: 5px;
  overflow: hidden;
  grid-area: display_name;
  white-space: nowrap;
  text-overflow: ellipsis;
  display: flex;
  justify-content: space-between;
  width: 100%;
  flex-wrap: nowrap;
}
.flex-name {
  max-width: calc(100% - 60px);
  overflow: hidden;
  text-overflow: ellipsis;
}
.user {
  cursor: text;
  font-size: 1.1rem;
}
.sml {
  font-size: 0.8em;
}
.gray {
  color: var(--gray);
}

.area-toot {
  cursor: text;
  user-select: auto;
  margin-top: 5px;
  margin-bottom: 5px;
  margin-left: 5px;
  grid-area: toot;
  width:100%;
}

.area-date_via {
  text-align: right;
  grid-area: date_via;
}

.area-additional {
  cursor: text;
  user-select: auto;
  grid-area: additional;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
}
.acct-note p {
  white-space: nowrap;
  text-overflow: ellipsis;
  overflow: hidden;
}
.area-toot.acct-note p:not(:first-child) {
  display: none;
}
.area-toot.acct-note p:first-child:after {
  content: "...";
  color: var(--gray);
}

.area-actions {
  padding: 10px;
  margin: 0;
  top: -5px;
  position: relative;
  display: flex;
  justify-content: space-around;
  max-width: 100%;
  grid-area: actions;
}

.cbadge {
  display: inline-block;
  max-width: calc(100% - 50px);
  padding: 3px 7px;
  margin: 3px;
  font-size: 0.8em;
  margin-right: 5px;
  line-height: 1;
  color: #fff;
  text-align: center;
  white-space: nowrap;
  vertical-align: middle;
  background-color: #777;
  border-radius: 10px;
  overflow: hidden;
  white-space: nowrap;
  text-overflow: ellipsis;
  height: calc(0.8em + 8px);
  user-select: none;
}
</style>
