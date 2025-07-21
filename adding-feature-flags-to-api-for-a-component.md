# Adding Feature Flags to API for Component

In an effort to provide maximum API flexibility while minimizing the Component Signature, it is now recommended that a
feature_flags attribute is defined in the <Component>Info rather than individual supports_* attributes.
Discussion below is based on adding feature_flags without attempting to deprecate existing supports_* attributes.

## esphome/aioesphomeapi/aioesphomeapi/model.py
Note:  MediaPlayerEntityFeature is defined in home-assistant/core/homeassistant/components/media_player/const.py otherwise
it would make sense to define in model.py

Add the feature_flags attribute to the <Component>Info, for example:
```
class MediaPlayerInfo(EntityInfo):
    ...
    feature_flags: int = 0
```
## esphome/aioesphomeapi/aioesphomeapi/api.proto
Update the ListEntities<Component>Response to include the new attribute, for example in ListEntitiesMediaPlayerResponse,
the next available attribute number is 11:

```
message ListEntitiesMediaPlayerResponse {
  option (id) = 63;
  ...
  uint32 feature_flags = 11;
}
```
## esphome/aioesphomeapi/tests/test_model.py
Add the new attribute to the testing for the component, for example:
```
def test_media_player_supported_format_convert_list() -> None:
    """Test list conversion for MediaPlayerSupportedFormat."""
    assert MediaPlayerInfo.from_dict(
        {
            "supports_pause": False,
            ...
            "feature_flags": 0,
        }
    ) == MediaPlayerInfo(
        supports_pause=False,
        ...
        feature_flags=0,
    )
```
## esphome/esphome/components/api/api.proto
Update the ListEntities<Component>Response to include the new attribute, for example in ListEntitiesMediaPlayerResponse,
the next available attribute number is 11:

```
message ListEntitiesMediaPlayerResponse {
  option (id) = 63;
  ...
  uint32 feature_flags = 11;
}
```
## esphome/esphome/components/api/api_connection.cpp
Update the try_send_<component>_info method to set the feature_flags value:
```
uint16_t APIConnection::try_send_media_player_info(EntityBase *entity, APIConnection *conn, uint32_t remaining_size,
                                                   bool is_single) {
  auto *media_player = static_cast<media_player::MediaPlayer *>(entity);
  auto traits = media_player->get_traits();
  ...
  msg.feature_flags = traits.get_feature_flags();
  ...
}
```
## esphome/esphome/components/**/<component>.h
Add the <Component>Feature enumeration, for example in esphome/esphome/components/media_player/media_player.h:
For the MediaPlayer Component, using the HA media_player component as a guide, see
home-assistant/core/homeassistant/components/media_player/const.py
```
enum MediaPlayerFeature : uint32_t {
  PAUSE = 1,
  SEEK = 2,
  VOLUME_SET = 4,
  VOLUME_MUTE = 8,
  PREVIOUS_TRACK = 16,
  NEXT_TRACK = 32,

  TURN_ON = 128,
  TURN_OFF = 256,
  PLAY_MEDIA = 512,
  VOLUME_STEP = 1024,
  SELECT_SOURCE = 2048,
  STOP = 4096,
  CLEAR_PLAYLIST = 8192,
  PLAY = 16384,
  SHUFFLE_SET = 32768,
  SELECT_SOUND_MODE = 65536,
  BROWSE_MEDIA = 131072,
  REPEAT_SET = 262144,
  GROUPING = 524288,
  MEDIA_ANNOUNCE = 1048576,
  MEDIA_ENQUEUE = 2097152,
  SEARCH_MEDIA = 4194304,
};
```
Update the component header file traits class, again for example the MediaPlayer component:
```
class MediaPlayerTraits {
 public:
  ...
    uint32_t get_feature_flags() const {
    uint32_t flags = 0;
    flags != MediaPlayerEntityFeature.PLAY_MEDIA
      | MediaPlayerEntityFeature.BROWSE_MEDIA
      | MediaPlayerEntityFeature.STOP
      | MediaPlayerEntityFeature.VOLUME_SET
      | MediaPlayerEntityFeature.VOLUME_MUTE
      | MediaPlayerEntityFeature.MEDIA_ANNOUNCE
    if (this->supports_pause_) {
      flags |= MediaPlayerEntityFeature.PAUSE | MediaPlayerEntityFeature.PLAY
    }
    return flags;
  }
  ...
};
```
## home-assistant/core/homeassistant/components/esphome/<component>.py
Update the _on_static_info_update to use feature_flag if it is available
```
def _on_static_info_update(self, static_info: EntityInfo) -> None:
        """Set attrs from static info."""
        super()._on_static_info_update(static_info)
        flags = 0
        if self._static_info.feature_flags:
            flags |= self._static_info.feature_flags
        else:
            flags |= (
                MediaPlayerEntityFeature.PLAY_MEDIA
                | MediaPlayerEntityFeature.BROWSE_MEDIA
                | MediaPlayerEntityFeature.STOP
                | MediaPlayerEntityFeature.VOLUME_SET
                | MediaPlayerEntityFeature.VOLUME_MUTE
                | MediaPlayerEntityFeature.MEDIA_ANNOUNCE
            )
            if self._static_info.supports_pause:
                flags |= MediaPlayerEntityFeature.PAUSE | MediaPlayerEntityFeature.PLAY
        self._attr_supported_features = flags
        self._entry_data.media_player_formats[self.unique_id] = cast(
            MediaPlayerInfo, static_info
        ).supported_formats
```


