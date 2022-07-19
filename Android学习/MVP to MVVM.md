# MVP的设计
从Andorid开发的角度，我们初始进行开发时，会做的是把逻辑都写在Activity中，就比如一个新闻页面
```
public class NesActivity extends AppCompatActivity {
    private NewsAdapter newsAdapter;
    @BindView(R.id.news_list)
    RecyclerView newsView;
    @Override
    protected void onCreate(Bundle savedInstance) {
        super.onCreate();
        setContentView(R.layout.news);
        initView();
        fetchNews();
    }

    private void initView() {
        newsAdapter = new NewsAdapter();
        newsView.setLayoutManager(new LinearLayoutManager(this, Vertical, false));
        newsView.setAdapter = newsAdapter;
    }

    public void fetchNews() {
        Retrofit retrofit = new Retrofit.Builder().baseUrl("https://domain/path").build();
        GetNewsApi api = retrofit.create(GetNewsApi.class);
        api.getNews().enqueue(new Callback<NewsInfo>() {
            @Override
            public void onResponse(Call<NewsInfo> call, Response<NewsInfo> response) {
                if (response != null && response.body() != null) {
                    NewsInfo info = response.body();
                    newsAdapter.setDataList(info.getNewsList());
                    newsAdapter.notifyDataSetChanged();
                }
            }

            @Override
            public void onFailure(Call<NewsInfo> call, Throwable t) {
                showFailView();
            }
        })
    }
}
```
这种方式得到的activity比较臃肿，会充斥着各种数据请求以及根据数据请求结果渲染ui的逻辑。如果我们需要替换数据获取的框架，或者修改数据获取的逻辑，都需要对Activity代码进行修改，这会导致Activity代码变得十分冗杂。即使我们把数据请求单独抽离，形成单独的model模块，依然面临着在Activity中编写各种回调代码的问题。

基于上述的问题，MVP架构应运而生，抽离出的中间层Presenter，来处理数据获取请求和ui渲染请求。即activity作为ui层，会发起数据获取的请求给presenter，presenter处理请求并转发给model层，model层请求数据，然后回调presenter根据结果请求渲染ui，activity处理渲染ui请求。修改后上述代码：

```
public class NesActivity extends AppCompatActivity implements INewsView {
    private NewsAdapter newsAdapter;
    @BindView(R.id.news_list)
    RecyclerView newsView;
    INewsPresenter mPresenter;
    @Override
    protected void onCreate(Bundle savedInstance) {
        super.onCreate();
        setContentView(R.layout.news);
        initView();
        mPresenter = new NewsPresenter(this);
        fetchNews();
    }

    private void initView() {
        newsAdapter = new NewsAdapter();
        newsView.setLayoutManager(new LinearLayoutManager(this, Vertical, false));
        newsView.setAdapter = newsAdapter;
    }

    public void fetchNews() {
        mPresenter.fetchNews();
        /**
            Retrofit retrofit = new Retrofit.Builder().baseUrl("https://domain/path").build();
            GetNewsApi api = retrofit.create(GetNewsApi.class);
            api.getNews().enqueue(new Callback<NewsInfo>() {
                @Override
                public void onResponse(Call<NewsInfo> call, Response<NewsInfo> response) {
                    if (response != null && response.body() != null) {
                        NewsInfo info = response.body();
                        newsAdapter.setDataList(info.getNewsList());
                        newsAdapter.notifyDataSetChanged();
                    }
                }

                @Override
                public void onFailure(Call<NewsInfo> call, Throwable t) {
                    showFailView();
                }
            })
        */
    }

    @Override
    public void showNews(boolean isShowFail, List<News> newsList) {
        newsAdapter.setDataList(info.getNewsList());
        newsAdapter.notifyDataSetChanged();
    }
}

public class NewsPresenter implements INewsPresenter {
    INewsView mView;
    INewsModel mRepo;
    public NewsPresenter(INewsView view) {
        mView = view;
        mRepo = new NewsRepo(this);
    }

    @Override
    public void fetchNews() {
        mRepo.fetchNews();
    }

    @Override
    public void showView(boolean isShowFail, List<News> newsList) {
        mView.showNews(isShowFail, newsList);
    }
}

public class NewsRepo implements INewsModel {
    INewsPresenter mPresenter;
    public NewsRepo(INewsPresenter presenter) {
        mPresenter = presenter;
    }

    @Override
    public void fetchNews() {
        Retrofit retrofit = new Retrofit.Builder().baseUrl("https://domain/path").build();
        GetNewsApi api = retrofit.create(GetNewsApi.class);
        api.getNews().enqueue(new Callback<NewsInfo>() {
            @Override
            public void onResponse(Call<NewsInfo> call, Response<NewsInfo> response) {
                if (response != null && response.body() != null) {
                    NewsInfo info = response.body();
                    mPresenter.showView(false, info.getNewsList());
                }
            }

            @Override
            public void onFailure(Call<NewsInfo> call, Throwable t) {
                mPresenter.showView(true, null);
            }
        })
    }
}
```
到这里，mvp的基本架构就写完了，确实解决了activity臃肿的问题。不过，随着越来越多的view和数据类之间产生交互，presenter就需要定义越来越多的接口，又会导致presenter变得很臃肿。而这种问题产生的根源，在于数据和视图之间的绑定关系需要由presenter决定，因此双向绑定的mvvm应运而生。

mvvm中，viewModel代替了presenter的角色，并通过liveData实现了数据到视图的双向绑定。view层监听livedata变化，model层改变livedata，这样避免了在presenter层中过疯狂定义接口，而只需要新增对应的liveData就能满足需要。