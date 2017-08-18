.PHONY: css clean

css: clean
	sass --style compressed sass/site.css.sass ../static/css/site.css
	# sass --style expanded --debug-info sass/site.css.sass ../static/css/site.css
	# sass --style expanded --debug-info --watch ./sass ../static/css
	# lessc less/reset.less ../static/css/reset.css
	# lessc less/pygments.less ../static/css/pygments.css
	# lessc less/main.less ../static/css/main.css

clean:
	rm -rf ../static/css
