<!--
  - Licensed to the Apache Software Foundation (ASF) under one or more
  - contributor license agreements.  See the NOTICE file distributed with
  - this work for additional information regarding copyright ownership.
  - The ASF licenses this file to You under the Apache License, Version 2.0
  - (the "License"); you may not use this file except in compliance with
  - the License.  You may obtain a copy of the License at
  -
  -     http://www.apache.org/licenses/LICENSE-2.0
  -
  - Unless required by applicable law or agreed to in writing, software
  - distributed under the License is distributed on an "AS IS" BASIS,
  - WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  - See the License for the specific language governing permissions and
  - limitations under the License.
  -->

<template>
  <v-container id="search" grid-list-xl fluid >
    <v-layout row wrap>
      <v-flex lg12>
        <v-card flat color="transparent">
          <v-card-text>
            <v-form>
              <v-layout row wrap>
                <v-combobox
                  id="serviceSearch"
                  :loading="loading"
                  :items="typeAhead"
                  :search-input.sync="input"
                  v-model="filter"
                  flat
                  append-icon=""
                  hide-no-data
                  :suffix="queryBy"
                  :hint="hint"
                  label="Search Dubbo Services"
                  @keyup.enter="submit"
                ></v-combobox>
                  <v-menu class="hidden-xs-only">
                    <v-btn slot="activator" large icon>
                      <v-icon>unfold_more</v-icon>
                    </v-btn>

                    <v-list>
                      <v-list-tile
                        v-for="(item, i) in items"
                        :key="i"
                        @click="selected = i">
                        <v-list-tile-title>{{ item.title }}</v-list-tile-title>
                      </v-list-tile>
                    </v-list>
                  </v-menu>
                  <v-btn @click="submit" color="primary" large>Search</v-btn>
              </v-layout>
            </v-form>

          </v-card-text>
        </v-card>
      </v-flex>
    </v-layout>

    <v-flex lg12>
      <v-card>
        <v-toolbar flat color="transparent" class="elevation-0">
          <v-toolbar-title><span class="headline">Search Result</span></v-toolbar-title>
          <v-spacer></v-spacer>
        </v-toolbar>

        <v-card-text class="pa-0">
          <template>
            <v-data-table
              hide-actions
              class="elevation-0 table-striped"
              :headers="headers"
              :items="services"
            >
              <template slot="items" slot-scope="props">
                <td >{{props.item.service}}</td>
                <td>{{props.item.group}}</td>
                <td>{{props.item.version}}</td>
                <td>{{props.item.appName}}</td>
                <td class="text-xs-center px-0"><v-btn small color='primary' :href='getHref(props.item.service, props.item.appName,
                                                                                            props.item.group, props.item.version)'>Detail</v-btn></td>
              </template>
            </v-data-table>
          </template>
          <v-divider></v-divider>
        </v-card-text>
      </v-card>
    </v-flex>
  </v-container>
</template>
<script>
  export default {
    data: () => ({
      items: [
        {id: 0, title: 'service name', value: 'serviceName'},
        {id: 1, title: 'IP', value: 'ip'},
        {id: 2, title: 'application', value: 'application'}
      ],
      timerID: null,
      loading: false,
      selected: 0,
      serviceItem: [],
      input: null,
      appItem: [],
      typeAhead: [],
      services: [],
      filter: '',
      headers: [
        {
          text: 'Service Name',
          value: 'service',
          align: 'left'
        },
        {
          text: 'Group',
          value: 'group',
          align: 'left'
        },
        {
          text: 'Version',
          value: 'version',
          align: 'left'
        },
        {
          text: 'Application',
          value: 'application',
          align: 'left'
        },
        {
          text: 'Operation',
          value: 'operation',
          sortable: false,
          width: '110px'
        }
      ]
    }),
    computed: {
      queryBy () {
        return 'by ' + this.items[this.selected].title
      },
      hint () {
        if (this.selected === 0) {
          return 'Service ID, org.apache.dubbo.demo.api.DemoService, * for all services'
        } else if (this.selected === 1) {
          return 'Find all services provided by the target server on the specified IP address'
        } else if (this.selected === 2) {
          return 'Input an application name to find all services provided by one particular application, * for all'
        }
      }
    },
    watch: {
      input (val) {
        this.querySelections(val)
      }
    },
    methods: {
      querySelections (v) {
        if (this.timerID) {
          clearTimeout(this.timerID)
        }
        // Simulated ajax query
        this.timerID = setTimeout(() => {
          if (v && v.length >= 4) {
            this.loading = true
            if (this.selected === 0) {
              this.typeAhead = this.serviceItem.filter(e => {
                return (e || '').toLowerCase().indexOf((v || '').toLowerCase()) > -1
              })
            } else if (this.selected === 2) {
              this.typeAhead = this.appItem.filter(e => {
                return (e || '').toLowerCase().indexOf((v || '').toLowerCase()) > -1
              })
            }
            this.loading = false
            this.timerID = null
          } else {
            this.typeAhead = []
          }
        }, 500)
      },
      getHref: function (service, app, group, version) {
        let query = 'service=' + service + '&app=' + app
        if (group !== null) {
          query = query + '&group=' + group
        }
        if (version != null) {
          query = query + '&version=' + version
        }
        return '/#/serviceDetail?' + query
      },
      submit () {
        this.filter = document.querySelector('#serviceSearch').value.trim()
        if (this.filter) {
          let pattern = this.items[this.selected].value
          this.search(this.filter, pattern, true)
        } else {
          return false
        }
      },
      search: function (filter, pattern, rewrite) {
        this.$axios.get('/service', {
          params: {
            pattern: pattern,
            filter: filter
          }
        }).then(response => {
          this.services = response.data
          if (rewrite) {
            this.$router.push({path: 'service', query: {filter: filter, pattern: pattern}})
          }
        })
      }
    },
    mounted: function () {
      let query = this.$route.query
      let filter = null
      let pattern = null
      let vm = this
      Object.keys(query).forEach(function (key) {
        if (key === 'filter') {
          filter = query[key]
        }
        if (key === 'pattern') {
          pattern = query[key]
        }
      })
      if (filter != null && pattern != null) {
        this.filter = filter
        if (pattern === 'serviceName') {
          this.selected = 0
        } else if (pattern === 'application') {
          this.selected = 2
        } else if (pattern === 'ip') {
          this.selected = 1
        }
        this.search(filter, pattern, false)
      }
      this.$axios.get('/service', {
        params: {
          pattern: 'serviceName',
          filter: '*'
        }
      }).then(response => {
        let length = response.data.length
        for (let i = 0; i < length; i++) {
          vm.serviceItem.push(response.data[i].service)
          vm.appItem.push(response.data[i].appName)
        }
      })
    }

  }
</script>

<style scoped>
  table.v-table tbody td {
    word-break: break-all;
  }

</style>

