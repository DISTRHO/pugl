config = configuration_data()
config.set('NAME', 'pugl_window_demo')

info_plist = configure_file(configuration: config,
                            input: files('../../resources/Info.plist.in'),
                            output: 'Info.plist')

subdir('MacOS')
