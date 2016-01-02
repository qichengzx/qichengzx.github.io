```
	/**
     * 错误处理
     *
     * @inner
     * @param {Object} response 返回的 JSON 数据
     * @param {Object=} errorHandler 自定义处理的错误类型
     * @return {Object}
     */
	function errorHandler(response, errorHandler) {

        var code = response.code;

        if (code !== 0) {

            var handler;

            if (errorHandler && (handler = errorHandler[code])) {
                if ($.isFunction(handler)) {
                    handler(response);
                }
            }
            else {

                var msg = errorCode[code] || response.msg;

                if (!msg) {

                    msg = [ ];

                    $.each(
                        response.data || { },
                        function (key, value) {
                            if (value) {
                                msg.push(value);
                            }
                        }
                    );

                    msg = msg.join('<br />');
                }

                if (msg && code != 2090102) {
                    alert({
                        content: msg,
                        width: 400
                    });
                }
            }
        }

        return response;
    }

    /**
     * 发送 post 请求
     *
     * @inner
     * @param {string} url 请求 url
     * @param {Object} data 发送的数据
     * @param {Object=} options
     * @property {boolean} options.sync 是否是同步请求
     * @property {Object=} options.errorHandler 自定义 error 处理
     *
     * @return {Promise}
     */
    function post(url, data, options) {

        options = options || { };
        data = data || { };

        /*var number = store.get('user').number || 0;

        data._user_number = number;

        $.extend(data, store.get('monkey'));*/

        return $.ajax({
            url: url,
            data: data,
            method: 'post',
            dataType: 'json',
            async: options.sync ? false : true
        })
        .pipe(function (response) {
            return errorHandler(response, options.errorHandler);
        });

    }
```