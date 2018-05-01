<template>
    <div class="container">
        <div class="row justify-content-center">
            <div class="col-md-8">
                <div class="card card-default">
                    <div class="card-header">NEO Wallet</div>
                    <transition name="fade">
                    <div v-if="!loggedin" class="card-body">
                        <input class="form-control" v-model="privateKey" placeholder="Private Key"/>
                        <input class="form-control" placeholder="Phrase" v-model="phrase" type="password">
                        <button v-on:click="createAccount()" class="btn btn-success">Login With Private Key</button>
                    </div>
                    <div v-else class="card-body">
                        <b>Address: </b>{{balance.address}}
                        <br>
                        <b>NEO: </b> {{parseInt(balance.assets.NEO.balance)}}
                        <br>
                        <b>GAS: </b> {{parseFloat(balance.assets.GAS.balance)}}
                        <br>
                        <ul>
                          <li v-for="token in tokens">{{token.symbol}}: {{parseFloat(token.balance)}}</li>
                        </ul>
                        <div class="form-group">
                          <button class="btn btn-sm btn-primary" v-on:click="sendAsset">Send</button>
                        </div>
                        <div class="form-group">
                          <button class="btn btn-sm btn-success" v-on:click="claimGas">Claim Gas</button>
                        </div>
                    </div>
                  </transition>
                </div>
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        mounted() {
            console.log('Component mounted.')
        },
        data(){
          return {
            Neon: {},
            account: null,
            wallet: null,
            balance: null,
            privateKey: '',
            phrase: '',
            loggedin:false,
            tokens: []
          }
        },
        created(){
          this.Neon = Neon.default;

        },
        methods:{
          createAccount:function(){
            this.account =new wallet.Account(this.privateKey).decrypt(this.phrase);
            var balance = new wallet.Balance({net: 'MainNet', address: this.account.address})
            var that = this;
            api.neonDB.getBalance('MainNet',this.account.address).then(obj => {
              that.balance = obj;
              const scriptHashes = [
                '34579e4614ac1a7bd295372d3de8621770c76cdc',
                'b951ecbbc5fe37a9c280a76cb0ce0014827294cf',
                'ecc6b20d3ccac1ee9ef109af5a7cdb85706b1df9'

              ];


              scriptHashes.forEach(function(item,index,array){
                var getName = { scriptHash:item, operation: 'name', args: [] }
                var getDecimals = { scriptHash:item, operation: 'decimals', args: [] }
                var getSymbol = { scriptHash:item, operation: 'symbol', args: [] }
                var getTotalSupply = { scriptHash:item, operation: 'totalSupply', args: [] }
                var getBalance = {
                  scriptHash: item,
                  operation: 'balanceOf',
                  args:[that.Neon.u.reverseHex(that.Neon.u.str2hexstring(that.account.address))]
                };
                console.log(item);
                var script = that.Neon.create.script([getName, getDecimals, getSymbol, getTotalSupply,getBalance]);
                Neon.rpc.Query.invokeScript(script)
                .execute('http://seed3.neo.org:20332')
                .then(res => {
                  console.log(res) // You should get a result with state: "HALT, BREAK"
                  if(res.result.state != "FAULT, BREAK"){
                    var token ={name:that.Neon.u.hexstring2str(res.result.stack[0].value), decimals: res.result.stack[1].value,symbol:that.Neon.u.hexstring2str(res.result.stack[2].value),
                    totalSupply:that.Neon.u.hexstring2str(res.result.stack[3].value),balance:that.Neon.u.hexstring2str(res.result.stack[4].value)};
                    that.tokens.push(token);
                  }

                });

              });
              that.loggedin = true;
            }).catch(err => {
              console.log(err);
            });


          },
          sendAsset: function(){
            console.log('sendAsset Clicked!');
          },
          claimGas: function(){

            this.Neon.claimGas({net: 'MainNet',address:this.account.address,privateKey: this.privateKey})
            .then(config => {
              console.log(config.response)
              alert('Gas CLAIMED!')
            })
            .catch(config => {
              console.log(config)
            })
          }
        }
    }
</script>
