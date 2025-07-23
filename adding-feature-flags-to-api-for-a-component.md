# Adding Feature Flags to API for Component

In an effort to provide maximum API flexibility while minimizing the Component Signature, it is now recommended that a
feature_flags attribute is defined in the <Component>Info rather than individual supports_* attributes.
Discussion below is based on adding feature_flags without attempting to deprecate existing supports_* attributes.

## esphome/aioesphomeapi/aioesphomeapi/model.py
Add the bitshift enumeration for the Features, for example for MediaPlayer, Note: MediaPlayerEntityFeature is originally
defined in home-assistant/core/homeassistant/components/media_player/const.py
```
class MediaPlayerEntityFeature(enum.IntFlag):
    """Supported features of the media player entity."""

    PAUSE = 1
    SEEK = 2
    VOLUME_SET = 4
    VOLUME_MUTE = 8
    PREVIOUS_TRACK = 16
    NEXT_TRACK = 32

    TURN_ON = 128
    TURN_OFF = 256
    PLAY_MEDIA = 512
    VOLUME_STEP = 1024
    SELECT_SOURCE = 2048
    STOP = 4096
    CLEAR_PLAYLIST = 8192
    PLAY = 16384
    SHUFFLE_SET = 32768
    SELECT_SOUND_MODE = 65536
    BROWSE_MEDIA = 131072
    REPEAT_SET = 262144
    GROUPING = 524288
    MEDIA_ANNOUNCE = 1048576
    MEDIA_ENQUEUE = 2097152
    SEARCH_MEDIA = 4194304
```
Update supports_* attributes to be deprecated.
Add the feature_flags attribute to the <Component>Info and add a compatability method,
Note: the APIVersion must be the current APIVersion at the time of merge into development, so APIVersion(2,3) must be updated to correct values.
```
class MediaPlayerInfo(EntityInfo):
    ...
    // Deprecated in API version 2.3
    bool supports_pause = 8 [deprecated=true];
    ...
    feature_flags: int = 0

    def feature_flags_compat(self, api_version: APIVersion) -> int
        if api_version < APIVersion(2, 3):
            flags = (
                MediaPlayerEntityFeature.PLAY_MEDIA
                | MediaPlayerEntityFeature.BROWSE_MEDIA
                | MediaPlayerEntityFeature.STOP
                | MediaPlayerEntityFeature.VOLUME_SET
                | MediaPlayerEntityFeature.VOLUME_MUTE
                | MediaPlayerEntityFeature.MEDIA_ANNOUNCE
            )
            if self.supports_pause:
                flags |= MediaPlayerEntityFeature.PAUSE | MediaPlayerEntityFeature.PLAY
            
            return flags

        return self.feature_flags
```
## esphome/aioesphomeapi/aioesphomeapi/api.proto
Update supports_* attributes to be deprecated.
Update the ListEntities<Component>Response to include the new attribute, for example in ListEntitiesMediaPlayerResponse,
the next available attribute number is 11:

```
message ListEntitiesMediaPlayerResponse {
  option (id) = 63;
  ...
  // Deprecated in API version 2.3
  bool supports_pause = 8 [deprecated=true];
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
Add a new test to make sure that compatibility function works before and after implementation
```
def test_media_player_feature_flags_compat() -> None:
    """Test feature flags works for before and after APIVersion implementation"""
    info = MediaPlayerInfo(
                   supports_pause=False,
                   supported_formats=[
                       MediaPlayerSupportedFormat(
                           format="flac",
                           sample_rate=48000,
                           num_channels=2,
                           purpose=1,
                           sample_bytes=2,
                       )
                   ],
                   #PLAY_MEDIA,BROWSE_MEDIA,STOP,VOLUME_SET,VOLUME_MUTE,MEDIA_ANNOUNCE
                   feature_flags=1184268,
               )
    assert info.feature_flags_compat(APIVersion(2, 2)) == info.feature_flags_compat(APIVersion(2, 3))
```
## esphome/esphome/components/api/api.proto
Make identical changes to esphome/aioesphomeapi/aioesphomeapi/api.proto
```
message ListEntitiesMediaPlayerResponse {
  option (id) = 63;
  ...
  // Deprecated in API version 2.3
  bool supports_pause = 8 [deprecated=true];
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
enum MediaPlayerEntityFeature : uint32_t {
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
    if (this->get_supports_pause()) {
      flags |= MediaPlayerEntityFeature.PAUSE | MediaPlayerEntityFeature.PLAY
    }
    return flags;
  }
  ...
};
```
## home-assistant/core/homeassistant/components/esphome/<component>.py
Update the _on_static_info_update to use method feature_flags_compat
```
def _on_static_info_update(self, static_info: EntityInfo) -> None:
        """Set attrs from static info."""
        super()._on_static_info_update(static_info)
        self._attr_supported_features = self._static_info.feature_flags_compat(self._api_version)
        self._entry_data.media_player_formats[self.unique_id] = cast(
            MediaPlayerInfo, static_info
        ).supported_formats
```
## home-assistant/core/tests/components/esphome/test_<component>.py

Update Tests as needed, for MediaPlayer anytime the supports_pause is set, also set
the feature_flags to the correct value, below is one example but there are 4 changes needed for MediaPlayer
```
...
        MediaPlayerInfo(
            object_id="mymedia_player",
            key=1,
            name="my media_player",
            supports_pause=True,
            #PLAY_MEDIA,BROWSE_MEDIA,STOP,VOLUME_SET,VOLUME_MUTE,MEDIA_ANNOUNCE,PAUSE,PLAY
            feature_flags=1200653,
        )
...
```
