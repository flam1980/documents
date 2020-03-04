##### 1.http上下文：

using http_context = std::unique_ptr<detail::http_context_impl, detail::http_context_deleter>;

context = eosio::client::http::create_http_context();

创建并持有http_context_impl(io_service)，和删除器http_context_deleter。





##### 2.构造命令树

eg：声明app存在子命令

app.require_subcommand();

*add_subcommand实现：* 

    /// Add a subcommand. Like the constructor, you can override the help message addition by setting help=false
    App *add_subcommand(std::string name, std::string description = "", bool help = true) {
        subcommands_.emplace_back(new App(description, help, detail::dummy));
        subcommands_.back()->name_ = name;
        subcommands_.back()->allow_extras();
        subcommands_.back()->parent_ = this;
        subcommands_.back()->ignore_case_ = ignore_case_;
        subcommands_.back()->fallthrough_ = fallthrough_;
        for(const auto &subc : subcommands_)
            if(subc.get() != subcommands_.back().get())
                if(subc->check_name(subcommands_.back()->name_) || subcommands_.back()->check_name(subc->name_))
                    throw OptionAlreadyAdded(subc->name_);
        return subcommands_.back().get();
    }
调用add_subcommand添加命令到命令树：

```
   // Push subcommand
   // 参数以app实例为根节点，创建第一层命令：
   auto push = app.add_subcommand("push", localized("Push arbitrary transactions to the blockchain"), false);
   //如果命令存在子命令，调用require_subcommand说明：
   push->require_subcommand();

   // push action
   string contract_account;
   string action;
   string data;
   vector<string> permissions;
   // 以一层命令为根，创建二层命令。
   auto actionsSubcommand = push->add_subcommand("action", localized("Push a transaction with a single action"));
   actionsSubcommand->fallthrough(false);
   actionsSubcommand->add_option("account", contract_account,
                                 localized("The account providing the contract to execute"), true)->required();
   actionsSubcommand->add_option("action", action,
                                 localized("A JSON string or filename defining the action to execute on the contract"), true)->required();
   actionsSubcommand->add_option("data", data, localized("The arguments to the contract"))->required();

   add_standard_transaction_options(actionsSubcommand);
   
   // 并设定该命令的回调函数。
   actionsSubcommand->set_callback([&] {
      fc::variant action_args_var;
      if( !data.empty() ) {
         try {
            action_args_var = json_from_file_or_string(data, fc::json::relaxed_parser);
         } EOS_RETHROW_EXCEPTIONS(action_type_exception, "Fail to parse action JSON data='${data}'", ("data", data))
      }
      auto accountPermissions = get_account_permissions(tx_permission);

      send_actions({chain::action{accountPermissions, contract_account, action, variant_to_bin( contract_account, action, action_args_var ) }});
   });
```

##### 3.调用回调函数

最后调用app.parse(argc, argv)分析命令行，并调用对应的回调函数：CLI11.hpp

```
    std::vector<std::string> parse(int argc, char **argv) {
        name_ = argv[0];
        std::vector<std::string> args;
        //从后向前入vector
        for(int i = argc - 1; i > 0; i--)
            args.emplace_back(argv[i]);
        return parse(args);
    }

    /// The real work is done here. Expects a reversed vector.
    /// Changes the vector to the remaining options.
    std::vector<std::string> &parse(std::vector<std::string> &args) {
        _validate();
        _parse(args);
        run_callback();
        return args;
    }
    
    /// Internal function to run (App) callback, top down
    递归遍历树，并调用所有存在的callback
    void run_callback() {
        pre_callback();
        if(callback_)
            callback_();
        for(App *subc : get_subcommands()) {
            subc->run_callback();
        }
    }
```



##### 4.method

![method](.\images\method.png)



##### 5.钱包状态

关闭keosd，钱包关闭

open：钱包打开，lock状态

lock/unlock：切换钱包上锁、开锁状态



##### 6.Action

   struct permission_level {
​      account_name    actor;			//授权账户名
​      permission_name permission;	//权限名称
   };

   struct action {
​      account_name               account;   //合约账户名
​      action_name                name;         //action名称
​      vector<permission_level>   authorization;  //授权该action的账户列表
​      bytes                      data;



##### 7.transaction

transaction_header  --> transaction  -->  signed_transaction  -->  deferred_transaction

packed_transaction：从signed_transaction构造

transaction_metadata：可以从signed_transaction或packed_transaction构造



##### 8.permission_object 可以看出权限是个树状结构，记录了shared_authority

```
class permission_object : public chainbase::object<permission_object_type, permission_object> {
      OBJECT_CTOR(permission_object, (auth) )

      id_type                           id;
      permission_usage_object::id_type  usage_id;
      id_type          parent; ///< parent permission 父权限
      account_name     owner; ///< the account this permission belongs to
      permission_name  name; ///< human-readable name for the permission
      time_point       last_updated; ///< the last time this authority was updated
      shared_authority auth; ///< authority required to execute this permission
```



##### 9.shared_authority   定义拥有该权限的account列表及对应的权重

```
struct shared_authority {
   uint32_t                                   threshold = 0;
   shared_vector<key_weight>                  keys;
   shared_vector<permission_level_weight>     accounts;
   shared_vector<wait_weight>                 waits;
```

```
struct key_weight {
   public_key_type key;
   weight_type     weight;
}
struct permission_level_weight {
   permission_level  permission;
   weight_type       weight;
}
struct wait_weight {
   uint32_t     wait_sec;
   weight_type  weight;
}
```



##### 10.start_block返回值，pending_block_mode

```
enum class start_block_result {
   succeeded,
   failed,
   waiting,
   exhausted
};

enum class pending_block_mode {
   producing,
   speculating
};
```



##### 11.块状态

```
enum class block_status {
irreversible = 0, ///< this block has already been applied before by this node and is considered irreversible
validated   = 1, ///< this is a complete block signed by a valid producer and has been previously applied by this node and therefore validated but it is not yet irreversible
complete   = 2, ///< this is a complete block signed by a valid producer but is not yet irreversible nor has it yet been applied by this node
incomplete  = 3, ///< this is an incomplete block (either being produced by a producer or speculatively produced by a node)
};
```



##### 11.block

```
struct block_header
{
  block_timestamp_type             timestamp;
  account_name                     producer;

  uint16_t                         confirmed = 1;  

  block_id_type                    previous;

  checksum256_type                 transaction_mroot; /// mroot of cycles_summary
  checksum256_type                 action_mroot; //mroot of all delivered action receipts


  uint32_t                          schedule_version = 0;
  optional<producer_schedule_type>  new_producers;
  extensions_type                   header_extensions;


  digest_type       digest()const;
  block_id_type     id() const;
  uint32_t          block_num() const { return num_from_id(previous) + 1; }
  static uint32_t   num_from_id(const block_id_type& id);
};

struct signed_block_header : public block_header
{
  signature_type    producer_signature;
};

struct signed_block : public signed_block_header {
private:
  signed_block( const signed_block& ) = default;
public:
  signed_block() = default;
  explicit signed_block( const signed_block_header& h ):signed_block_header(h){}
  signed_block( signed_block&& ) = default;
  signed_block& operator=(const signed_block&) = delete;
  signed_block clone() const { return *this; }

  vector<transaction_receipt>   transactions; /// new or generated transactions
  extensions_type               block_extensions;
};
```

##### 12.block_state

继承自block_header_state，增加了签名块block和事务列表trxs

```
struct block_state : public block_header_state {
  explicit block_state( const block_header_state& cur ):block_header_state(cur){}
  block_state( const block_header_state& prev, signed_block_ptr b, bool skip_validate_signee );
  block_state( const block_header_state& prev, block_timestamp_type when );
  block_state() = default;

  /// weak_ptr prev_block_state....
  signed_block_ptr                                    block;
  bool                                                validated = false;
  bool                                                in_current_chain = false;

  /// this data is redundant with the data stored in block, but facilitates
  /// recapturing transactions when we pop a block
  vector<transaction_metadata_ptr>                    trxs;
};
```



```
struct block_header_state {
    block_id_type                     id;
    uint32_t                          block_num = 0;
    signed_block_header               header;
    uint32_t                          dpos_proposed_irreversible_blocknum = 0;
    uint32_t                          dpos_irreversible_blocknum = 0;
    uint32_t                          bft_irreversible_blocknum = 0;
    uint32_t                          pending_schedule_lib_num = 0;//last irr block num
    digest_type                       pending_schedule_hash;
    producer_schedule_type            pending_schedule;
    producer_schedule_type            active_schedule;
    incremental_merkle                blockroot_merkle;
    flat_map<account_name,uint32_t>   producer_to_last_produced;
    flat_map<account_name,uint32_t>   producer_to_last_implied_irb;
    public_key_type                   block_signing_key;
    vector<uint8_t>                   confirm_count;
    vector<header_confirmation>       confirmations;

    block_header_state   next( const signed_block_header& h, bool trust = false )const;
    block_header_state   generate_next( block_timestamp_type when )const;

    void set_new_producers( producer_schedule_type next_pending );
    void set_confirmed( uint16_t num_prev_blocks );
    void add_confirmation( const header_confirmation& c );
    bool maybe_promote_pending();


    bool                 has_pending_producers()const { return pending_schedule.producers.size(); }
    uint32_t             calc_dpos_last_irreversible()const;
    bool                 is_active_producer( account_name n )const;

    producer_key         get_scheduled_producer( block_timestamp_type t )const;
    const block_id_type& prev()const { return header.previous; }
    digest_type          sig_digest()const;
    void                 sign( const std::function<signature_type(const digest_type&)>& signer );
    public_key_type      signee()const;
    void                 verify_signee(const public_key_type& signee)const;
};
```



##### 13.confirm_count数组

保存最近的1024个块需要的确认数量，随着块的创建滚动。

在set_confirmed时，会根据传入的参数，将对应块的confirm_count减一，减到0意味着块成为全网不可逆块的候选

    ////下个块的生产者将上个块产生的全网不可逆块（confirm_count计算2/3+1节点确认的块）加入候选列表
    result.producer_to_last_implied_irb[prokey.producer_name] = result.dpos_proposed_irreversible_blocknum;
    
    //从候选列表1/3高度处得到全网不可逆块号
    result.dpos_irreversible_blocknum = result.calc_dpos_last_irreversible();


```
block_header_state::set_confirmed {
    dpos_proposed_irreversible_blocknum = block_num_for_i;
}
```



##### 14.交易（transaction）总结

5.1 交易执行命令
5.2 交易执行过程
5.3 交易签名及作用
5.4 交易鉴权过程
5.5 交易到智能合约执行
5.6 交易打包
5.7 交易的发送
5.8 交易的验证
5.9 总结



##### 15.签名，签名校验

```
   using block_id_type       = fc::sha256;
   using checksum_type       = fc::sha256;
   using checksum256_type    = fc::sha256;
   using checksum512_type    = fc::sha512;
   using checksum160_type    = fc::ripemd160;
   using transaction_id_type = checksum_type;
   using digest_type         = checksum_type;

```

示例：

```

```

